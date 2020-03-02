+++
date = 2020-03-02T00:00:00Z
slug = "patching-xdebug-docker-for-mac"
tags = ["docker", "php"]
title = "Patching Xdebug performance issues on Docker for Mac"
+++

### Background on how Docker for Mac works
Docker is a Linux-only technology. Docker for Mac uses a small (very small) Linux VM to implement what "feels like" native Docker support on Mac OS. The VM is implemented using XHyve and MacOS's not-much-publicized native hypervisor support. The fact that Docker runs in a VM is ultimately the source of this issue, though that's not being entirely fair.

### Why is everything that has to do with time in computing hard?
The actual issue that we're dealing with here is that when your Mac sleeps, obviously programs stop executing and to some extent, time is suspended within the OS. When it wakes up again, you're generally oblivious to any complexity there because of smart people having solved this problem and because of time syncing tools like NTP.

_However_, because nothing is ever that easy, we run into some issues. When your computer sleeps, so does the VM that Docker uses. When that VM wakes up again, there are several complex things happening, but the basic story is that the Linux kernel inside the VM freaks out and doesn't know what time it is. This causes it to distrust its default system clock source (TSC, "Time Stamp Counter"), and fall back to another available time source. That time source, HPET ("High Precision Event Timer"), is much more reliable however it's also much, much slower to access. The time source that we want to be using in almost ever circumstance is TSC. This is a "userland" time source and while it can be affected by things like how busy your processor is, that is largely mitigated by NTP, which Docker for Mac uses to keep its VM in close sync with the host. More reading on this [here](https://blog.packagecloud.io/eng/2017/03/08/system-calls-are-much-slower-on-ec2/).

The annoying thing is that currently the only known resolution for this is to "reboot" Docker for Mac, which will cause the clock sync to be "fresh" and will let the VM trust the fast TSC clock source again. There does not appear to be any way to convince a running Docker for Mac VM to revert back to TSC once it has marked it as an unstable time source. Realistically, for people who pick their laptop up and close it and move around frequently, it's untennable to reboot every local container you have running every time you open your laptop.

There are several open issues on the topic on various Github repos ([Docker for Mac](https://github.com/docker/for-mac/issues/3455), [xhyve](https://github.com/machyve/xhyve/issues/36)).The challenge appears to be, as of this writing, that everyone is blaming an upstream project (not unreasonable in this case), to the point where the finger is left pointing at a lack of exposed APIs from MacOS itself.

This is also affecting other heavy users of `gettimeofday` on Docker for Mac. `EXPLAIN` on Postgres is a [notable example](https://benchling.engineering/analyzing-performance-analysis-performance-56cb2e212629)

### So how does this affect Xdebug?
One of the useful things Xdebug does is collect timing data. Even if you're not profiling, Xdebug is making calls out to `gettimeofday()` constantly during your script execution so that it can provide that information on the stack trace output in case an exception is thrown. During an average Laravel request, this means Xdebug is making thousands upon thousands of calls out to the OS to get the current time. This is normally not _much_ of a problem except in in the scenario outlined above, where suddenly getting the current time becomes much, much more expensive.

In my brief testing, this can add more than one order of magnitude to the time it takes to execute a script, depending on how deep the stack is, to the point where many requests that might take about 1 second on my laptop are starting to hit the 30 second `max_execution_time`. 

This issue was [raised](https://bugs.xdebug.org/view.php?id=1701) on the Xdebug issue tracker, and Derick said, I think rightly so, that he was not interested in applying yet another configuration flag for Xdebug to support something that is clearly a Docker problem. 

### Patching Xdebug
Luckily for those who are interested in a short or medium term fix, Xdebug already has one central wrapper for its timing calls so we can hack this in one function and it'll be solved across the board. 

*Note, I would consider this an "advanced manuever". You need to be comfortable compiling C code in order to proceed, though for most people on most systems, this should "just work".*

We can start by checking out a function called `xdebug_get_utime` in what is aptly named `usefulstuff.c` ([source here](https://github.com/xdebug/xdebug/blob/master/src/lib/usefulstuff.c)). We can see below that this function is already aware of some differences in what time functions might be available and has a macro that might cause this function to return `0` in some scenarios. This is a good sign, because it means the rest of Xdebug must be generally OK with returning 0 here. 

```c
double xdebug_get_utime(void)
{
#ifdef HAVE_GETTIMEOFDAY
	struct timeval tp;
	long sec = 0L;
	double msec = 0.0;

	if (gettimeofday((struct timeval *) &tp, NULL) == 0) {
		sec = tp.tv_sec;
		msec = (double) (tp.tv_usec / MICRO_IN_SEC);

		if (msec >= 1.0) {
			msec -= (long) msec;
		}
		return msec + sec;
	}
#endif
	return 0;
}
```

Initially I was going to just hijack that existing macro and define it to false no matter what, but I realized that appears to be coming from `time.h` and I didn't want to inadvertently break something beyond my understanding elsewhere in the compile chain. So, I decided to fork the repo and just delete all the code within that `#ifdef` statement. After the changes, this function now looks like:

```c
double xdebug_get_utime(void)
{
	return 0;
}
```

My fork of Xdebug is available here: [jimbojsb/xdebug](https://github.com/jimbojsb/xdebug). You're welcome to clone and compile this version if you want, though I can't promise that I'll be diligent about keeping it up to date with the upstream repo, as my use case won't see me compiling this very often. It would probably be better to maintain your own copy if you want to keep up to date on your own terms. 

### Compiling and installing our changes
I'm going to show how to compile this in a `Dockerfile` as I can't imagine why you'd care about any of this otherwise. My most commonly used base image is `phusion/baseimage` which is Ubuntu-based. If you're using another flavor like Alpine, your commands will vary, primarily around what's required to get a working build environment.

###### Step 1: Build and compile Xdebug in isolation
```Dockerfile
FROM phusion/baseimage
RUN apt-add-repository -y ppa:ondrej/php \
    && install_clean \
        pkg-config \
        php7.3-dev \
        git

WORKDIR /root
RUN git clone https://github.com/jimbojsb/xdebug.git
WORKDIR /root/xdebug
RUN phpize && ./configure && make
```
Here we're using the very common `ondrej/php` PPA to install the latest PHP development package, as well as git and `pkg-config` which is usually needed when compiling PHP from source. This example is PHP 7.3 as that happens to be where what environment I'm using currently for the apps that are most affected by this issue, but there should be no changes required for PHP 7.4. 

Note that we're not running `make install`. My intention here is to use Docker multi-stage builds in order to "pluck" the output of the compile process (`xdebug.so`) into the actual Docker image that I'm using. If you don't want to do that, perhaps if you already are committed to having a full build toolchain in your Docker image, you could just include these steps in your existing Dockerfile and run `make install` and be done with it. 

To reference the output of this Docker build in another image, we should tag this with something easy to remember during the build. My build command looks like this: `docker build -t php-xdebug -f Dockerfile-xdebug .`

###### Step 2: Pluck xdebug.so into our real Docker image
Once you've successfully build the Dockerfile as outlined above, you'll then want to use the output in your actual app's Dockerfile. We can do that by using and advanced `COPY` command, as below:

```Dockerfile
COPY --from=php-xdebug /root/xdebug/modules/xdebug.so /usr/lib/php/20180731/xdebug.so
```

The parts that are important here are the `--from=` and the destination inside `/usr/lib`. The `--from=` value must match what you tagged your Xdebug build image above as. The destination in `/usr/lib` will vary, you may need to inspect your Docker image to find out where PHP extensions are expected to live. If you're using `ondrej/php` the above location will be correct for PHP 7.3, but will need to be updated with the appropriate API version (the date part of the path) depending on which major release of PHP you're using. If you're using Alpine or some other Linux flavor, this may look entirely different.

We're just plucking the Xdebug extension itself here, I didn't bother copying any of the stock `xdebug.ini` extension configuration files as I already had a custom one in place and didn't need it. You could certainly copy that over as well if you need it. You can also keep installing Xdebug from the PHP package source that you normally use and just overwrite the `xdebug.so` file later in your Docker build.

### Wrap Up
This fix renders the time issues on Docker for Mac a non-issue for Xdebug purposes. It doesn't fix other time related problem (for example, if your PHP script is making thousands of timing calls itself). It is also kneecapping XDebug's ability to time things, so just know that is a trade off here. I wouldn't trust any profiling data generated by Xdebug if you apply this patch.

Is this ideal? No. However the performance issues caused by this problem are also not acceptable, so this might be a workable strategy for some until this problem is fixed at the Docker for Mac level. I was not willing to work without Xdebug so this was the compromise I came up with. I've also offered to contribute a compile-time flag to Xdebug that would make this change happen, but I'm skeptical that would be accepted, and I mostly agree with that stance. 

As part of this exercise I also experimented with a workflow where I completely disable Xdebug from PHP when I'm not actively going to be using it. I believe for heavy projects, this is a worthwhile thing to do. If you're doing work like changing HTML templates or some other not-likely-step-debug task, you'll get even more performance by just turning it Xdebug off. I'll probably end up combining both approaches in my projects.


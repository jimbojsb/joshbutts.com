+++
date = 2020-05-13T00:00:00Z
slug = "tableplus-shortcuts"
tags = ["docker", "bash", "mac"]
title = "Advanced TablePlus usage from the command line on MacOS"
+++
![Traefik](https://tableplus.com/resources/images/workspace@2x.png
)

### Foreword
I spent probably about half of my professional career as a hardcore Navicat user. Navicat was an actual *good* SQL GUI (MySQL specifically) that worked on Windows. When I later moved to MacOS, they had a version there too, though it never really felt as polished as the Windows version. Navicat still exists, but for whatever reason I transitioned to Sequel Pro ("Pancakes"). I used that for many years, but alas, it has become somewhat abandonware. 

 Enter [TablePlus](https://tableplus.com/). I learned that this thing even existed from Taylor Otwell on Twitter. It's modern, kept up to date, supports every database you'd ever credibly want to connect to, and pretty to look at. Yes, it costs money, but often the good things in life do. I've happily bought a license for it and for anyone on my team who wants to use it. Some of them still insist on using DataGrip, which I cannot understand.

### artisan db:open
A few months back, Caleb Porzio [tweeted a Gist](https://gist.github.com/calebporzio/20b7d71b8b2690c788f28070f00edcbc) about how to use the `open` command on MacOS and a custom Artisan command to read your Laravel DB config and create a proper `mysql://` link that would cause TablePlus to open your local dev database without having to set up a connection. This very interesting, but I wanted more, as evidenced by my comment that remains on the Gist to this day.

### What is "more"
For starters, my teams all exclusively use Docker and Docker Compose for local development. This means, in short, that the PHP we have installed on our macs, if it even actually is installed, isn't guaranteed to be a good enough environment to invoke any given Laravel project's artisan command. We use the in-container PHP binaries for anything involving PHP work. 

So, with that restriction in mind, Caleb's `artisan db:open` wouldn't quite work, but an equivalent shell function would. Turns out someone already provided the baseline version of that on the Gist. However, I wanted to solve for two more things:  

1. I really really wanted to achieve the "color coding" and "environment tagging" open shell-opened TablePlus windows. I have come to rely on this visual cue to know whether I have a local or production connection open. Since TablePlus seemed to support url-style arguments for some of what Caleb had done, I figured it was possible, but found it to be undocumented.


2. My teams run dozens of Laravel projects, and I was growing tired of keeping an agreed upon list of database ports so that we could run multiple projects and connect to their DBs without having any port collisions. Docker can auto-assign a high-numbered non-colliding port but its not predictable, and so it would be a pain to managed saved database connections. I figured I could read the current state from my version of the function and pick out the port that had been assigned, that way it would always be correct and we could ditch the static port mappsings altogether.

### My Version

```bash
function tp {
	PORT=$(docker-compose port mysql 3306 | cut -d: -f2)
	NAME=$(basename $(pwd))
	DATABASE=$(cat docker-compose.yml | grep MYSQL_DATABASE | cut -d: -f2 | xargs)
	open "mysql://root:root@127.0.0.1:$PORT/$DATABASE?Enviroment=local&Name=$NAME&statusColor=DAEBC2"
}
```

The above bash function was my version of how to achieve this. I'm using some basic bash-fu and dockuer-compose features to achieve this. You almost certainly don't want my very specific version of this, but this should be enough to show you the how to concatenate a URL together based on how you want to detect the database information.

The key pieces of the URL format are as follows:

* **[scheme]://**: I'm using `mysql://` but I tested a few other obvious ones and they work (`postgres://`, `redis://`)
* **[username]:[password]@**: These are passed using standard HTTP-style colon-separated-followed-by-@ credentials (also the same as JDBC-style)
* **[hostname]**: self explanatory I hope...
* **:[port]**: not required if you're using 3306 (running locally, only have one project, etc). If you've mapped some other high-numbered port via Docker or similar, you need to pass it after the host
* **/[database]**: these vary slightly per server type, but for MySQL, this is the database name on the server. It's optional if you want to not pre-select one.
* **Enviroment=**: Yes there is a typo in this query string key name. This is analgous to the dropdown labeled "Tag" in the connection editor in the GUI. Choices are `local`, `testing`, `development`, `staging`, `production`. This does not seem to accept any other values. This value is color coded in the UI independently of anythign else, and shows at the right hand side of the connection status bar at the top of the window.
* **Name=**: This option will show on the left hand side of the connection status bar, to the left of the database name. This is the equivalent of the "Name" field in the connection editor window
* **statusColor=**: This is an RGB hex string (6 hex characters, all caps, no leading #). This can be any hex value. Since I use the stock green color in the connection editor window for local development connections, I pulled that color out with an eye dropper and determined it to be `DAEBC2`. 

So, the final url that one might build for a local dev database that was MySQL might look like this:
`mysql://username:password@hostname:portnumber/databasename?Enviroment=local&Name=local&statusColor=DAEBC2`

### Wrap Up
Hopefully TablePlus documents these things and my blog post on it becomes irrelevant in the future. Until then, if you're as picky about everything being "just so" as I am, you know have full control over TablePlus "dynamic" connections from whatever command-line context you like.
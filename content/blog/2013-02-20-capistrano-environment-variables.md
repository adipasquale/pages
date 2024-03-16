---
title: Capsitrano Environment variables
date: 2013-02-20
categories: en
tags: dev rails
---

This is the story of an epic fight. Me (regular guy) vs the server (Rails 3, deployed via Capistrano to a Passenger – Nginx hosted Ubuntu server, using Mandrill transactional mail service). not a fair fight.

If you already know the deal (or just don’t want to hear me whine) jump [there](#task) for the Capistrano task that’ll magically expand your environment variables in the config.

First if you follow Mandrill’s [guide](https://help.mandrill.com/entries/21738467-Using-Mandrill-s-SMTP-integration-with-Web-Frameworks) (spoiler: you should not), this is what you’d set up in your config/production.rb :

```ruby
config.action_mailer.smtp_settings = {
  :address   => "smtp.mandrillapp.com",
  :port      => 587,
  :enable_starttls_auto => true,
  :user_name => "MANDRILL_USERNAME",
  :password  => "MANDRILL_API_KEY",
  :authentication => 'login'
}
```

I know your extra picky security cerebrum already picked up the issue with this. hardcoding the username and password in your config file cannot be a good idea, especially if you’re sharing your code with many people. A better way is to use environment variables.

```ruby
:user_name => ENV["MANDRILL_USERNAME"],
:password  => ENV["MANDRILL_API_KEY"],
```

you then have to set these variables on your prod server, in /etc/profile for example :

```shell
export MANDRILL_USERNAME=foo
export MANDRILL_API_KEY=bar
```

Now, it can’t be that simple, right ? right.
Whereas my mails were not sent, the logger did not give me any errors, so first things first, I figured I’d turn the failed mail deliveries warning on in config/production.rb :

```ruby
config.action_mailer.raise_delivery_errors = true
```

if you’re doing this live on your production server (and you should) you have to restart Passenger :

```shell
touch tmp/restart.txt
```

And I could now see a beautiful `Net::SMTPServerBusy, Relay access denied error`. First I believed it came from a blocked 587 port, got postfix to work, and everything worked fin, hmmm. Heck, it even worked when I launched a thin server on the prod !
This [Stack Overflow](https://stackoverflow.com/questions/13963795/rails-mailer-netsmtpserverbusy) entry showed me the way to the problem : Passenger doesn’t set your environment variables when you fire it up.

One solution is to load your variables in the wrapper around Ruby interpreter that Passenger uses (more info on this [here](https://blog.rayapps.com/2008/05/21/using-mod_rails-with-rails-applications-on-oracle/)). I don’t like this idea much, it feels very hacky, what’ll happen when I’ll upgrade my Ruby for example ?

My solution is to expand the variables in your config file during the deploy (meaning replacing ENV[“MANDRILL_USERNAME”] with the actuall username value, copied from the environment variable).
This amazing [SO answer](https://stackoverflow.com/questions/1609423/using-sed-to-expand-environment-variables-inside-files#answer-1610500) helped a lot, I tweaked it a bit to fit a Capistrano deploy task. It will expand ANY of your environment variables, how great is that ?!

Warning: this is some mad-ass syntax, even the built-in highlighter can’t figure it out. There’s a LOT of escaping, between sed syntax and ruby’s …

```ruby
desc "Replace environment variables with hardcoded values in config file"
task :replace_env_vars, roles: :app do
  run "mv #{release_path}/config/environments/production.rb #{release_path}/config/environments/production.before_sed.rb"
  run 'env | sed \'s/[\%]/\\&amp;/g;s/\([^=]*\)=\(.*\)/s%ENV\\\[\\\"\1\\\"\\\]%\"\2\"%/\' > ' + "#{release_path}/script/expand_env_vars.sed.script"
  run "cat #{release_path}/config/environments/production.before_sed.rb | sed -f #{release_path}/script/expand_env_vars.sed.script > #{release_path}/config/environments/production.rb"
end

after "deploy:update_code", "deploy:replace_env_vars"
```

Note : this will only replace strings that use double quotes like `ENV[“JESSICA_ALBA_PHONE_NUMBER”]` (I gather you wouldn’t share that piece of intel with any other developer)

Hey, lucky you ! There’s a last step, and it’s so easy it feels good. By default, the SSH session opened by Capistrano doesn’t execute your init scripts (~/.profile, ~/.bashrc, not even /etc/profile/). These are [DanM instructions](https://pretheory.wordpress.com/2008/02/12/capistrano-path-and-environment-variable-issues/) (cheers !) to load them up. Create `~/.ssh/environment` and reput the export instructions, or you can even load your whole `/etc/profile` file, even though I have no idea what the security implications are. just don’t if you are ignorant like me.
Tell your SSH server to load this file : add this line in `/etc/ssh/sshd_config` :

```
PermitUserEnvironment yes
```

and restart it with

```shell
sudo /etc/init.d/ssh restart
```

That should make your day. If it does not, feel free to let out your sorrow in the comments.

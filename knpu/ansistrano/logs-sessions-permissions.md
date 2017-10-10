# Logs, Sessions & File Permissions

Let's tackle one of the most *confusing* thing in Symfony: how to handle file
permissions for the cache directory. Even I couldn't figure this out for years!

To get our site working, we're setting the *entire* `var/` directory to 777. This
includes `cache/`, `logs/` and `sessions/`.

This is a bummer for security. Here's my big question: after we deploy, which files
*really* needs to be writable?

Let's solve an ancient Symfony mystery. To start, instead of setting the entire `var/`
directory to 777, let's *just* set this for `var/logs`. This is actually the reason
we originally created this task: our site was failing because `var/logs` wasn't writable.

But first, back in `deploy.yml`, create a new variable: `release_logs_path` set to
`{{ ansistrano_shared_path }}/var/logs`. `ansistrano_shared_path` is a special variable
that Ansistrano gives us. Thanks!

Copy the variable, and back in `after-symlink-shared`, use it.

Oh, and we don't need `follow` anymore. But *do* add `become: true`. Why? The files
in this directory - like `prod.log` - will probably be created by the web server,
so, `www-data`. The `become: true` will allow us to change those permissions.

Ok, let's try this! Find your local terminal, and deploy!

```terminal-silent
ansible-playbook ansible/deploy.yml -i ansible/hosts.ini --ask-vault-pass
```

When this finishes, *only* `var/logs` should be writable.

Deep breath. Refresh! Ah dang, it fails! That's ok! Let's play detective and find
the problem.

## Using Native PHP Sessions

Back on the server, find the `var/log` directory and tail `prod.log`. Oh!

> Unable to create the directory `var/sessions`

Apparently the `var/sessions` directory needs to be writable so that the session
data can be stored.

But wait! Before we make that writable, I have a better solution. Open up
`app/config/config.yml`. Look under `framework` and `session`. Ah! *This* is the
reason why sessions are stored in `var/sessions`. Change that: set `handler_id`
to `~`. I'll add a comment: this means that the default PHP session handler will
be used.

Why are we doing this? Well, PHP *already* knows how to handle and store sessions.
It will find a directory on the file system to store them and *it* will handle permissions...
because making them 777 isn't a great idea. This will be the default config for
new Symfony 4 projects.

Go back to the local terminal. We just made a change to our *code*, so we need to
commit and push. Now, deploy!

```terminal-silent
ansible-playbook ansible/deploy.yml -i ansible/hosts.ini --ask-vault-pass
```

An even *better* session setup - especially if you want to move servers or use multiple
servers - is to store them somewhere else, like the database or Memcache. You can
find details about that in the Symfony docs. That's what we do for KnpU.

Ok! Let's try it again.. refresh! It works! OMG, it's alive! So... does this mean
that the `var/cache` directory does *not* need to be writable? Well... not so fast.
Go back to the server. Move up a few directories and into current. Check out the
`var/cache/prod` directory:

```terminal-silent
ls -l var/cache/prod
```

Woh! The cache files are writable by everyone! And so *of course* the site is working!
But... we didn't set the cache directory to 777? So, what's going on?

We still have two unanswered questions. First, why the heck is `var/cache/prod` writable
by everyone? And second, if we make it *not* writable, will our site still work?

Let's solve these mysteries next.
# Certbot instructions (flawed)

The Mastodon instructions ask you to enable a `nginx`
server with certain `ssl` features enabled (but not the keys).

You'll run into some trouble there (chicken and egg problem).

You cannot do `certbot --nginx -d example.com` with the as-is
`nginx` configuration.


## The fix.

Do not apply the HTTPS additions to the `nginx` config yet.

Just run `certbot certonly -d example.com`

Acquire the keyfiles.

Then put the keyfiles where they belong (see the Mastodon HTTPS
template for clear guidance)

THEN you can add the HTTPS `nginx` config, and THEN startup nginx

## Test


Then you can re-run `certbot --nginx -d example.com` to verify that
it all was cool.

Nothing will get changed (if you so desire), but that will help.

I do not know why Mastodon has the HTTPS template for `nginx` include the
SSL options --- it is a pure chicken and egg situation -- unless I'm
missing something.

Maybe I am?




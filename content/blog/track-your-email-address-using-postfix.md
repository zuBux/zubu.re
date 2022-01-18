+++
draft = false
date = "2015-11-16T16:03:15Z"
title = "Track who's leaking your e-mail address using Postfix"

+++

Curious about who is selling your e-mail address to spammers?

Most of us prefer to keep our material world contact information private. We usually share our phone number or home address with people we trust. However, in order to use most web services we are required to provide an e-mail address. Meanwhile the trustworthiness of a service is determined long after it gets hold of this info.

Surprisingly, e-mail addresses are a precious commodity. They are bought and sold everyday. They get leaked due to poor website design or dumped from hacked websites, along with the rest of your personal information. Is there a way to determine who leaked/sold your address? Considering the fact that we use the same e-mail address again and again, this can be nearly impossible unless you have a unique address for every single account you register (and spammers are aware of tricks like this). This is one of the many perks in hosting your own mail server.

Consider the following senario. You own `ihatespam.com` and `mail.ihatespam.com` MX record points to your Postfix mail server. Your precious personal e-mail address is `johndoe@ihatespam.com` but you don't want to give it away to every website you visit. Furthermore, you would like to know who is responsible for all the spam you receive.

You can achieve this by using unique non-existent e-mail addresses and redirect every e-mail you receive to those non-existent addresses to your inbox. Just add the following to ``/etc/postfix/main.cf`:

```
luser_relay = johndoe   #Use your username/mail account here
local_recipient_maps =
```

and reload your Postfix configuration

`service postfix reload`

The above configuration creates a catch-all destination for unknown recipients, instead of returning messages as undeliverable, which is the default behavior.

Now you can use unique e-mail addresses (e.x *john_lnkdn@ihatespam.com*, *john_fcbk@ihatespam.com*, *john_twitr@ihatespam.com*) to track any shady use of your personal information without having to manage multiple inboxes.

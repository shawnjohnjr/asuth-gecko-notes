## General Overview ##

### Bind Requests ###

The GET requests to https://mail.google.com/mail/u/0/channel/bind?OSID=..
seem to be long-polls that intermittenly result in 400 responses that look like
this:
```
<HTML>
<HEAD>
<TITLE>Unknown SID</TITLE>
</HEAD>
<BODY BGCOLOR="#FFFFFF" TEXT="#000000">
<H1>Unknown SID</H1>
<H2>Error 400</H2>
</BODY>
</HTML>
```

At least some of the time, the 400 responses occur immediately.  The long-polls
frequently seem to be 200 responess that contain noop payloads.  Specifically:

(This is from a long poll when things are working.)
```
17
[[8,["noop"]
]
]
17
[[9,["noop"]
]
]
18
[[10,["noop"]
]
]
18
[[11,["noop"]
]
]
18
[[12,["noop"]
]
]
18
[[13,["noop"]
]
]
18
[[14,["noop"]
]
]
18
[[15,["noop"]
]
]
18
[[16,["noop"]
]
]
```

Which came immediately after this POST's response, which explains the sequence
numbers.
```
268
[[0,["c","DF0DB44246696EC2","b",8]
]
,[1,["b"]
]
,[2,["c",["p",1,"","","",0,"",[],0,0]]
]
,[3,["c",["ud","Andrew Sutherland","Andrew Sutherland","Andrew",0]]
]
,[4,["c",["cse2",1]]
]
,[5,["c",["p",1,"","","",0,"",[],0,0]]
]
,[6,["c",["ef",1]]
]
,[7,["c",["cu",0]]
]
]
```

There was then a subsequent GET that continued the nops, 17-20.  It was preceded
by

### General Page Navigation ###

This seems to happen via GET and POST responses to
https://mail.google.com/mail/u/0/?ui=2

These all include responses that start with `)]}'` and followed by a blank line,
then the actual conent that is a nested thing starting with `[[[`

The ordering seems to be GET followed by POST.  The GET responses seem
consistent across multiple navigations for me.:
```
)]}'

[[["ci",[]],["cm",[],[]]],"c0ab49e094aad1e4"]
```

But the POST changes.

###  ###


18
[[18,["noop"]
]
]
653
[[19,["n",["n","[\"n_nm\",0,0,\"15d80105bf5dfd4b\",\"15d80105bf5dfd4b\",\"15d80105bf5dfd4b\",[\"^io_unim\",\"^unsub\",\"^cob-processed-gmr\",\"^fmas\",\"^i\",\"^sq_ig_i_promo\",\"^all\",\"^smartlabel_promo\",\"^cob_sm_emailmessage\",\"^u\",\"^esa\",\"^cob_goldmine\",\"^us\",\"^os_promo\"]\n,\"Our Select Mini 3D Printer is #1 | Thank You Special (White): $199.99 Free Sh...\",\"Thank you for helping us become #1. Can\\u0026#39;t see this Email? View it in your Browser monoprice_logo #1 Selling Model Printer Worldwide MP Select Mini 3D Printer V2, White $199.99 $219.99 Save 9% Free\",\"Monoprice\",\"monoprice@news.monoprice.com\",[]\n]\n",[]]]
]
]
18
[[20,["noop"]
]
]

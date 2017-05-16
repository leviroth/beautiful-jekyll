---
layout: post
title: Tips and Tricks for Reddit's New Modmail API
---

This document collects various quirks and useful pieces of information that I
came across in adding support for the new modmail API
to [PRAW](http://praw.readthedocs.io/en/latest/).

This was originally published
on
[GitHub Gist](https://gist.github.com/leviroth/dafcf1331737e2b55dd6fb86257dcb8d).

# Secret Codes #

Certain pieces of data are represented by numerical codes, which had to be
deciphered through trial and error. These lists could easily be incomplete.

## Mod actions ##

Thanks
to
[/u/creesch](https://www.reddit.com/r/redditdev/comments/5jf4yg/api_new_modmail/dbhyf0t/) for
finding these.

Code | Meaning
--- | ---
0 | highlight
1 | un-highlight
2 | archive
4 | "reported to admins" (reserved for future use)
3 | un-archive
5 | mute
6 | un-mute

## Conversation states ##

Code | Meaning
--- | ---
0 | new
1 | inprogress
2 | archived

# Endpoints #

## General ##

For `state` parameters, `all` does not include conversations which have been archived or mod conversations. In other words, it shows conversations that appear in the "All Modmail" tab on Reddit's web interface.

## Specific endpoints ##

### `POST /api/mod/bulk_read` ###

This endpoint is actually at `POST /api/mod/conversations/bulk/read`.

The `entity` parameter is required, and can't be `'all'`. So, if you want to
mark every conversation as read, you need to list all subreddits you moderate
which are using the new modmail.

### `GET /api/mod/conversations` ###
 
The default `sort` is `recent`, and the default `state` is `all`. `limit` can be
very high; I got up to 1998 before I ran out of modmails with which to test.

### `POST /api/mod/conversations` ###

According to documentation, the `to` parameter is the "Modmail conversation
recipient fullname." In fact, it is the display name.

### `POST /api/mod/conversations/:conversation_id` ###

Even for a private moderator note (i.e. with `isInternal` set to `true`),
`isAuthorHidden` can be set to either `true` or `false`. This apparently makes
no functional difference (since moderators can see hidden authors, and only
moderators can see the message), but it makes a cosmetic difference: the
crossed-out silhouette icon next to the author's name will be toggled.

### `POST /api/mod/conversations/:conversation_id/mute` ###

The response looks just like the response from `GET
/api/mod/conversations/:conversation_id`, except that the `conversation` key is
named `conversations` instead.

### `GET /api/mod/conversations/:conversation_id/user` ###

This doesn't appear to return any information that isn't already contained in
`GET /api/mod/conversations/:conversation_id/user`.

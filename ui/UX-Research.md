# UX Research and Requirements

The discover and define phase for the music-db web app: who it's for, what they're trying to do, and what v1 must let them do. The screens that answer these needs are in [UI-Design.md](../UI-Design.md).

There hasn't been any user research yet. Everything here is a **hypothesis** drawn from the product idea, and should be checked with the tests in [Assumptions to test](#assumptions-to-test).

**Contents**

- [Problem statement](#problem-statement)
- [Goals and non-goals](#goals-and-non-goals)
- [Personas](#personas)
- [Jobs to be done](#jobs-to-be-done)
- [User stories](#user-stories)
- [Success measures](#success-measures)
- [Assumptions to test](#assumptions-to-test)

---

## Problem statement

People who love music collect it in playlists, but on most platforms a profile is a long, unsorted pile of every playlist they ever made. Visitors can't tell what someone is really about. Collecting is also slower than it should be: adding a song means hunting through a long list of playlists every time.

**How might we** let people collect songs into playlists quickly, and show who they are through **five** playlists they chose, theirs or anyone else's?

The limits are the product: 5 favorites make a profile a curated statement, 20 songs keep playlists focused, and 20 playlists keep a collection manageable.

---

## Goals and non-goals

| Goals for v1 | Non-goals for v1 |
|---|---|
| Adding a song to a playlist takes two taps from wherever the song is. | Playing music. |
| A profile shows what someone is about at a glance. | Search, and complete browsing of the catalog. |
| People discover playlists made by others. | Social features beyond favoriting (following people, comments, a feed). |
| Each limit is clear before you reach it, and reaching one never loses work. | Editing a playlist's details or song order, private playlists, collaborators. |

---

## Personas

### Ana, the collector (primary)

- 34, Mexico City. Owns 9 playlists, *Road Trip* is her favorite. Adds songs a few at a time, usually on her phone.
- **Wants:** to drop a song into the right playlist without thinking, and to have her profile show off *Road Trip* and *Boleros para mamá*.
- **Frustration:** "Every app makes me scroll through all my playlists to add one song, and the one I want is always the one I used last."
- **What it means for the design:** the add menu puts **last used** first. Her favorites lead Home.

### Kenji, the explorer

- 27, Osaka. Makes few playlists himself, but likes finding other people's.
- **Wants:** a stream of interesting playlists, and a way to keep the best ones.
- **Frustration:** "Recommendations always show me what I already like."
- **What it means for the design:** Discover is random, not tuned to taste. A favorite takes one tap, and the limit of 5 makes him choose.

### Diego, the sharer

- 52, Guadalajara. Makes playlists for his family and sends the link on WhatsApp.
- **Wants:** to send a link that just works, even for people with no account.
- **Frustration:** "My mother can't open things that ask her to log in."
- **What it means for the design:** a public playlist's page is readable while signed out, and Copy link is one button. Only favorites are public, so Diego keeps the family playlists he wants to send among his 5 favorites.

---

## Jobs to be done

| When… | I want to… | So I can… |
|---|---|---|
| I come across a song I like | add it to the right playlist in one move | keep collecting without breaking my flow. |
| I open the app with no plan | see my favorites and a few new things | pick up where I was, or find something new. |
| I want to show my taste | pick exactly which 5 playlists represent me | let people understand me in seconds. |
| I find someone's great playlist | keep it with my own favorites | come back to it and show it on my profile. |
| I make a playlist for someone | send a link | they can open it right away. |

---

## User stories

Each story has acceptance criteria that tests and design reviews can check. The **Screen** column links to the spec.

| ID | As… | I want to… | Acceptance criteria | Screen |
|---|---|---|---|---|
| US-01 | a new visitor | create an account | Name, username, email and password are required, and country is optional. The username is suggested from the name. Errors appear on the field they're about. After sign-up I'm logged in and on Home. | [S2](../UI-Design.md#s2-sign-up) |
| US-02 | a returning user | log in | Username or email, plus password. A wrong combination gives one message that doesn't say which part was wrong. After login I return to the page I was trying to open. | [S1](../UI-Design.md#s1-login) |
| US-03 | a user | see my favorites first on Home | Up to 5, A–Z by name, with empty slots up to 5 and a count "n of 5". | [S3](../UI-Design.md#s3-home) |
| US-04 | a user | see random songs and add them | Home shows 6 songs; Songs shows 20 with Shuffle and Load more. Each has Add. | [S3](../UI-Design.md#s3-home), [S4](../UI-Design.md#s4-songs) |
| US-17 | a user looking for a particular song | search for it | The Songs screen has a search box. From 2 characters, matches by title, artist or album load as I type, ignoring accents and case. Each result has Add. ✕ or Esc brings back the random list. No matches gives a message with a hint. | [S4](../UI-Design.md#s4-songs) |
| US-05 | a user adding a song | find the playlist fast | The add menu shows my last used playlist first, then all my playlists A–Z, then "New playlist". Each shows "n/20". Full playlists are disabled and say "Full". Playlists that already have the song say "✓ Added". | [S4](../UI-Design.md#s4-songs) |
| US-06 | a user who added the wrong song | undo it | The toast after adding has Undo for 5 seconds. | [S4](../UI-Design.md#s4-songs) |
| US-07 | a user | create a playlist from anywhere | "New playlist" is at the top of every page. The dialog asks only for a name (required, up to 120 characters) and a description (optional), and says the playlist will be private. After creating, I'm on the new playlist. | [S5](../UI-Design.md#s5-create-playlist) |
| US-08 | a user adding a song with no fitting playlist | create one from the add menu | "+ New playlist" in the menu creates the playlist with that song in it, and I stay where I was. | [S5](../UI-Design.md#s5-create-playlist) |
| US-09 | a user with 20 playlists | know why I can't make another | "New playlist" is disabled and explains the limit and how to free a slot. | [S5](../UI-Design.md#s5-create-playlist) |
| US-10 | a user | discover other people's playlists | Discover shows random public playlists (other people's favorites) that I haven't favorited, with Shuffle and Load more. | [S6](../UI-Design.md#s6-discover-playlists) |
| US-11 | a user | favorite a playlist | One tap on the heart, on a card or on the playlist page. It works on my own playlists too. The count "n of 5" updates. | [S6](../UI-Design.md#s6-discover-playlists), [S7](../UI-Design.md#s7-playlist) |
| US-12 | a user with 5 favorites | favorite another | A dialog lists my 5 favorites A–Z so I can pick one to replace. My own playlists are marked "becomes private". Cancel changes nothing. | [Replace a favorite](../UI-Design.md#replace-a-favorite) |
| US-13 | anyone | open a public playlist | The name in large type, then the owner (a link to their profile), the description, counts, and the list of songs. It works signed out. A private playlist that isn't mine shows "We can't find this playlist". | [S7](../UI-Design.md#s7-playlist) |
| US-14 | a playlist owner | share it | On a public playlist, Copy link puts its URL on the clipboard and confirms it. On a private one the button is disabled and says how to make it public. | [S7](../UI-Design.md#s7-playlist) |
| US-15 | a playlist owner | remove a song, or delete the playlist | Remove is on each row, with Undo. Delete asks to confirm and says it can't be undone. | [S7](../UI-Design.md#s7-playlist) |
| US-16 | anyone | see a person's profile | Picture, name in large type, then their favorites A–Z. Favorites are always public, so everyone sees the same five. My own profile also shows empty slots and the count. | [S8](../UI-Design.md#s8-profile-me-and-others) |
| US-18 | a playlist owner | edit its name and description | "Edit details" on my playlist opens the form filled in. Save is disabled until something changes. After saving, the page shows the new details. | [Edit details](../UI-Design.md#edit-details) |
| US-19 | a playlist owner | control who sees my playlist | My playlists are private until someone favorites them; favoriting my own makes it public and puts it on my profile. The playlist page says "Public · on your profile", "Public · in 2 people's favorites" or "Private · only you". Unfavoriting takes it off my profile; it becomes private only when nobody else has it, so the people who favorited it keep it. | [S7](../UI-Design.md#s7-playlist) |
| US-20 | a user removing someone else's playlist from my favorites | know if I'll lose access | When I'm the only person who has it, I'm told it will become private and I won't be able to open it again, and I confirm first. | [Last favorite](../UI-Design.md#last-favorite) |

---

## Success measures

Measured once the app runs, through product analytics (the tool isn't chosen yet).

| Measure | Target for v1 |
|---|---|
| Time from tapping Add to the song being added (median) | under 3 seconds |
| Adds that use the **last used** item | over 40% for people with 3 or more playlists |
| Sign-ups that favorite at least one playlist in the first week | over 50% |
| People who reach 5 favorites and then replace one | tracked; a sign the limit makes people choose |
| Sign-up form errors per completed sign-up | under 1 |

---

## Assumptions to test

| ID | Assumption | Test before or during v1 |
|---|---|---|
| A-01 | People want **last used** first, not only A–Z. | 5 hallway tests with the mockup's add menu: count how often the target is the last used playlist. |
| A-02 | A limit of **5 favorites** feels like curation, not a restriction. | Ask 5 people to set up their profile in the mockup. Watch the Replace dialog. Ask what they think of the limit. |
| A-03 | Random recommendations are interesting enough without taste-based ranking. | Track favorites per Discover visit after launch. Compare with a simple "most favorited" list. |
| A-04 | People understand that "favorite" means "show on my profile". | First-click test: "Make Road Trip appear on your profile." |
| A-05 | Not playing music in v1 is acceptable. | Ask in testing. Watch for people tapping song rows expecting playback. |
| A-06 | People understand that favoriting their own playlist is what makes it public, and that it stays public while others have it as a favorite. | First-click test: "Make Sunday Morning visible to your friends." Then: "Hide it again." Count wrong first clicks. |
| A-07 | Searching by title, artist or album is enough; people don't need filters (genre, year) in v1. | Watch search sessions: how often a search returns no results, or is retyped. |

# User Flows

The main v1 flows, step by step, including what happens when a limit is hit or a request fails. Screen IDs (S1–S8) and rule IDs (BR-xx) refer to [UI-Design.md](../UI-Design.md). GitHub renders the diagrams.

**Contents**

- [F1 Sign up and log in](#f1-sign-up-and-log-in)
- [F2 Add a song to a playlist](#f2-add-a-song-to-a-playlist)
- [F3 Create a playlist](#f3-create-a-playlist)
- [F4 Favorite a playlist (limit of 5)](#f4-favorite-a-playlist-limit-of-5)
- [F5 Share a playlist](#f5-share-a-playlist)
- [F6 Remove a song or delete a playlist](#f6-remove-a-song-or-delete-a-playlist)
- [F7 Search for a song](#f7-search-for-a-song)
- [F8 Edit a playlist's details](#f8-edit-a-playlists-details)

---

## F1 Sign up and log in

```mermaid
flowchart TD
    A([Open the app]) --> B{Signed in?}
    B -- yes --> H[S3 Home]
    B -- no --> L[S1 Login]
    L -- "Create an account" --> S[S2 Sign up]
    S -- "Already have an account?" --> L

    S --> S1[Fill in name, username, email, password, country]
    S1 --> S2{Client checks pass?}
    S2 -- no --> S3[Show the error on its field] --> S1
    S2 -- yes --> S4["POST /users"]
    S4 -- "409 username_taken / email_taken" --> S3
    S4 -- 201 --> S5["POST /auth/login"] --> H

    L --> L1["POST /auth/login"]
    L1 -- "401 invalid_credentials" --> L2[Same message for unknown user and wrong password] --> L
    L1 -- 200 --> R{Came from a link?}
    R -- yes --> P[That page]
    R -- no --> H
```

The access token lives 15 minutes. The app refreshes it in the background (`POST /auth/refresh`) and sends the person to Login, keeping the page they were on, only when the refresh fails.

---

## F2 Add a song to a playlist

The flow that matters most. It starts on Home ("Songs for you"), on S4 Songs (random songs), or on S4 search results ([F7](#f7-search-for-a-song)).

```mermaid
flowchart TD
    A[Tap + Add on a song] --> M[Open the add menu]
    M --> M1["Last used: the playlist I added to most recently (BR-09)"]
    M --> M2["All playlists A–Z, each showing n/20"]
    M --> M3[+ New playlist]

    M1 --> C{Playlist state}
    M2 --> C
    C -- "Full: 20/20" --> X1[Row disabled, says Full]
    C -- "Already has the song" --> X2[Row disabled, says Added]
    C -- available --> D[Optimistic: count +1, menu closes, toast 'Added to Road Trip · Undo']
    D --> E["POST /playlists/{id}/tracks with If-Match"]
    E -- 201 --> F[Keep. Remember this playlist as last used]
    E -- 412 --> G[Get the version again, retry once]
    G -- 201 --> F
    G -- fails --> Z[Undo the count. Toast: 'Couldn't add the song. Try again']
    E -- "422 playlist_full" --> Y[Mark it Full. Toast explains and suggests another playlist]

    F --> U{Undo within 5 s?}
    U -- yes --> U1["DELETE /playlists/{id}/tracks/{position}"]
    U -- no --> End([Done])

    M3 --> F3[[F3 Create a playlist, with this song]]
```

---

## F3 Create a playlist

```mermaid
flowchart TD
    A{Where from?}
    A -- "Header: New playlist" --> B{Owns fewer than 20? BR-02}
    A -- "Add menu: + New playlist" --> B
    B -- no --> B1[Button disabled with the reason: delete one to make room]
    B -- yes --> C[S5 dialog: name has focus]
    C --> D[Type a name, optionally a description]
    D --> E{Name has a non-space character?}
    E -- no --> E1[Create stays disabled]
    E -- yes --> F["POST /playlists, visibility private (BR-10), plus track_ids when it came from the add menu"]
    F -- 201 --> G{Where from?}
    G -- header --> G1[Go to S7, the new empty playlist]
    G -- add menu --> G2[Stay. Toast 'Created Sunday Morning and added Come Together']
    F -- "422 playlist_limit_reached" --> B1
    F -- 422 name --> D
    C -- "Esc or ✕" --> H{Anything typed?}
    H -- no --> Close([Close])
    H -- yes --> H1[Ask: Discard this playlist?] --> Close
```

---

## F4 Favorite a playlist (limit of 5)

Starts from a heart on S3 Home, S6 Discover, S7 Playlist or S8 Profile. Favorites also decide visibility: a playlist is public while **anyone** has it as a favorite, and private when nobody does (BR-12, D-10).

```mermaid
flowchart TD
    A[Tap the heart] --> B{Already a favorite?}

    B -- yes --> L{Am I its last favorite?}
    L -- "no: others have it too" --> U1["DELETE /me/following/playlists/{id}: stays public"]
    U1 --> T1{My own playlist?}
    T1 -- yes --> T1a["Toast 'Road Trip left your profile. It stays public while 2 people have it' · Undo"]
    T1 -- no --> T1b["Toast 'Removed Night Drive from favorites' · Undo"]
    L -- yes --> O{My own playlist?}
    O -- yes --> U2["DELETE: it becomes private"] --> T2["Toast 'Road Trip is now private' · Undo"]
    O -- no --> LF["Last favorite dialog: 'it becomes private and you can't open it again'"]
    LF -- Cancel --> N([Nothing changes])
    LF -- Remove --> U3["DELETE: private, only its owner can open it"]

    B -- no --> C{"Favorites < 5? (BR-04)"}
    C -- yes --> D["PUT /me/following/playlists/{id}: public now, if it wasn't"]
    D --> D0{My own playlist?}
    D0 -- yes --> D1["Toast 'Road Trip is now public and on your profile'"]
    D0 -- no --> D2[Heart filled, count +1]
    D -- "422 favorite_limit_reached (another tab)" --> R
    C -- no --> R["Replace a favorite dialog: my 5, A–Z, each says if it stays public or becomes private"]
    R -- Cancel --> N
    R -- "Pick one + Replace" --> S["PUT /me/following/playlists/{new} with replaces: old, one transaction"]
    S --> S1["Toast 'Replaced Sunday Morning with Night Drive'"]
```

Own playlists and other people's count toward the same 5. Favorites appear A–Z on Home ("Your favorites") and on the profile, and everyone who opens the profile sees all of them (BR-06). Nobody loses a favorite because of what someone else does: only deleting a playlist removes it from other people's favorites.

---

## F5 Share a playlist

```mermaid
sequenceDiagram
    actor Diego
    participant App
    actor Mother as Diego's mother
    Diego->>App: S7 Playlist → Copy link
    App-->>Diego: Toast "Link copied" (https://app/playlists/22)
    Diego->>Mother: Sends the link in a chat
    Mother->>App: Opens the link, not signed in
    App-->>Mother: S7 readable: name, owner, songs
    Mother->>App: Taps the heart
    App-->>Mother: "Log in to favorite playlists" (Log in · Sign up)
```

Only public playlists (ones that someone has as a favorite) can be shared, so the plain URL works for anyone. On a private playlist, Copy link is disabled with the hint "Private playlists can't be shared. Favorite it to make it public." Share links for private playlists (`unlisted` + `share_token`) are a future extension.

---

## F6 Remove a song or delete a playlist

```mermaid
flowchart TD
    A[S7, owner] --> B{What?}
    B -- "Row ⋯ → Remove from playlist" --> C[Row disappears. Toast 'Removed Money · Undo']
    C --> D["DELETE /playlists/{id}/tracks/{position} with If-Match"]
    D -- 412 --> E[Reload the list: 'This playlist changed. We've reloaded it.']
    C -- Undo --> F["POST /playlists/{id}/tracks with position: put it back where it was"]
    B -- "Header ⋯ → Delete playlist" --> G[Confirm: 'Delete Road Trip? Its 7 songs are removed from it. This can't be undone.']
    G -- Cancel --> H([Nothing changes])
    G -- Delete --> I["DELETE /playlists/{id}"] --> J[Go to Home. Toast 'Deleted Road Trip'. Playlist count -1]
```

If the deleted playlist was a favorite, it leaves the favorites too (the database cascade removes its follows), so a slot opens. That includes other people's favorites. When other people have it, the confirmation says so: "Delete Road Trip? 2 people have it in their favorites." (Q-09.)

---

## F7 Search for a song

On S4 Songs (D-08).

```mermaid
flowchart TD
    A[Type in the search box] --> B{2 characters or more?}
    B -- no --> R[The random list stays]
    B -- yes --> W[Wait 300 ms after the last key, or Enter]
    W --> C[Cancel the previous search. Put the query in the URL: /songs?q=…]
    C --> D["GET /search?q=…&types=track&limit=20 (G-12)"]
    D -- results --> E["'9 songs match daft punk' + song rows with + Add"]
    D -- none --> F["'No songs match …' with a hint"]
    D -- error --> G["'Search isn't working right now. Try again.'"]
    E -- "+ Add" --> F2[[F2 Add a song to a playlist]]
    E -- "Load more results" --> D2["Same search, offset +20"] --> E
    E -- "✕, Esc or Shuffle" --> R
```

---

## F8 Edit a playlist's details

On S7, owner only (D-09).

```mermaid
flowchart TD
    A["Edit details (button or ⋯ menu)"] --> B[Dialog filled with the current name and description]
    B --> C{Changed, and the name isn't blank?}
    C -- no --> C1[Save stays disabled]
    C -- yes --> D["PATCH /playlists/{id} with the changed fields + If-Match"]
    D -- 200 --> E[Close. The page shows the new details. Toast 'Saved']
    D -- 412 --> F[Reload the values: 'This playlist changed while you were editing…'] --> B
    D -- 422 --> G[Show the error on the name field] --> B
    B -- "Esc or ✕ with changes" --> H[Ask: Discard your changes?]
```

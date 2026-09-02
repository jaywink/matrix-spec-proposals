# MSC0000: Template for new MSCs

This MSC builds on the legacy and sliding sync MSC's for profile updates.

This MSC aims to provide a solution for Matrix clients to fetch user profile data from Matrix servers.
Currently, there are multiple ways a client may end up receiving profile data from the backend.

1) By fetching a full profile or individual fields using `GET /_matrix/client/v3/profile/{userId}`
  or `GET /_matrix/client/v3/profile/{userId}/{keyName}`.
2) From legacy and sliding sync responses.

Additionally, display names and avatar urls may be discovered by clients using other methods. For 
this MSC, we're assuming the client would also need to know about custom profile fields.

The first option above is clear - the client knows it needs a profile, and it fetches it. Or it might
just want a single field, and can fetch it when needed. However, this method is inefficient if the
client needs to fetch multiple profiles.

The second option, profiles data down sync streams works for many cases. As per the MSC's involved,
clients will receive profile data in roughly the following ways:

* For initial sync, full profiles always for members in the rooms the sync relates to.
* For incremental sync:
  * Updates to profile fields 
  * When lazy loading, full profiles if the user has events in the sync timeline

What is not covered by profile updates via sync streams is the following situation:

* Alice, on a lazy loading client, syncs
* Alice joins a room
* Alice does a lazy loading incremental sync

In this situation, Alice would receive the full profiles of any users who had timeline events in the
lazy loading sync response. Alice's client would not receive profile information regarding users in
the joined room that the client didn't know about before, unless those users had events in the timeline.

## Proposal

Just as we already have a `GET /_matrix/client/v3/rooms/{roomId}/members`, this MSC proposes a new endpoint
`GET /_matrix/client/v3/rooms/{roomId}/profiles`. Clients that use the members endpoint to populate
the full members list for rooms in lazy loading sync, could then proceed to also populating the profiles
of any room members that profiles have not been seen for in the client. This may be important for
clients that want to show for example a user status of users in the room membership list.

## Client-Server API Changes

### Get room profiles

- **Endpoint**: `GET /_matrix/client/v3/rooms/{roomId}/profiles`
- **Description**: Retrieve room member user profiles.
- **Pagination**: *Yes? What mechanism?*
- **Response**:

```json
{
  "@alice:example.com": {
    "displayname": "Alice",
    "m.status": {
      "emoji": "💬",
      "text": "In a meeting"
    }
  },
  "@bob:example.com": {
    "displayname": "Bob"
  }
}
```

Optional parameters:

* `members` - A query parameter to fetch only particular members, passed in as comma separated full
  user identifiers. Users in the request must be found as members in the room. Pagination cannot be
  used if fetching members directly.
  For example;

  `GET /_matrix/client/v3/rooms/{roomId}/profiles?members=@alice:example.com,@bob:example.com`

TODO: error codes, etc.

## Potential issues

Passing in a `members` parameter for a lot of members may produce very long URLs. Clients would need
to ensure they fetch profiles in batches.

## Alternatives

### Provide full profiles in sync based on the user joining a room

It would be possible to solve missing profiles in clients for users joining new rooms in lazy loading mode
by just delivering all the room member profiles to the client every time the syncing user joins a new room.
This would be a change to the legacy and sliding sync MSCs on profile updates. This solution is not very
elegant and would potentially cause a massive amount of traffic to clients. This solution does not
give the control to the client on what users profiles they want to fetch.

### Extend the room members endpoint to return profiles

Extending the `GET /_matrix/client/v3/rooms/{roomId}/members` endpoint, which clients might use
to fetch the full room membership list to return profiles would ensure clients don't need to
make an extra endpoint call. This could, for example be, a new `profiles` top level key, to
avoid polluting the returned `ClientEvent` objects in `chunk`.

### Extend the room joined members endpoint to return profiles

Extending `GET /_matrix/client/v3/rooms/{roomId}/joined_members` to return more profile information
than just `display_name` and `avatar_url`, would allow for clients to use this endpoint to fetch
profile information. However, this endpoint is not paginated, and thus could produce extremely large
responses for large rooms, as the data stored in profiles may outweigh the current display names
and avatar urls by many factors.

## Security considerations

None foreseen at this moment.

## Unstable prefix

While this MSC is unstable, the endpoint is `GET /_matrix/client/unstable/org.matrix.mscxxxx/rooms/{roomId}/profiles`,
advertised via the `org.matrix.mscxxxx` flag in `/_matrix/client/versions`.

## Dependencies

This MSC builds on MSCxxxx, MSCyyyy and MSCzzzz (which at the time of writing have not yet been accepted
into the spec).

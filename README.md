# justCTF 2023 RPG game

This is the source code for the RPG game from the 2023 edition of justCTF.

There are two challenges in this game:
- Trial of Data - analyze the compiled game data in order to complete 5 challenges
- Trial of Bugs - two challenges that require the player to find bugs with the game

// TODO FIXME: include videos from game

## Hints

Here are some hints that were released during the challenge.

Set of hints for Trial of Bugs all the way to the solution - Room 1:
- the RPC is weird... see the @bothExecute decorator
- the tp to the room doesn't use the normal teleportation function, but instead directly sets the player coordinates - you could've noticed this if your ping was non-zero
- the RPC doesn't immediately execute the function, instead it sends a packet to the client and enqueues the actual function execution
- the client can freely delay RPCs originating from the server (serverside RPCs wait for the client to ACK them, the client "controls" the timeline), while still sending input/update RPCs; however all RPCs are executed when disconnecting, so the client can only delay their execution, not cancel them
- the room is split into 4 chunks, you need to let the game load only 3 out of the 4 chunks by stopping ACKing the server-side RPCs at the right moment; after you get past the barriers just refresh the page

Room 2: 
- collision bug
- if you press left while dropping the card it doesn't get immediately picked up when you collide with it again
- you need to align the card at the correct position near the bottom right corner of the walkable area while pressing left on the keyboard; then you can just walk right on the very bottom part of the room and it'll just let you pass through; you can write a script to make the alignment easier but it's doable without any scripts

## Solution

The game had a custom RPC implementation where the client has control over the "timeline". Basically the client can invoke RPCs whenever it wants, but if the servers wants to start some operation on it's own it needs to send a request to the client - the server-originated RPC is not invoked until either the client ACKs it or disconnects. loadChunk was one of the operations that used server-originated RPCs - the client could delay chunk loading for as long as it wanted. The room with the challenge was split into 4 chunks, so you needed to let the first 3 chunks load, and the 4th chunk had an obstacle that spanned multiple chunks (but persistent entities were only saved in the chunk where it's center was), so it disappeared when the fourth chunk was not loaded and the player could pass through.

## Development - no database

In order to develop this game, you need the following software:
- [Tiled](https://www.mapeditor.org/)
- [Spright](https://github.com/houmain/spright)
- some image editor of your choice

In order to build the game for the first time you need to do these steps:
- build Spright and edit the local_config.json file with the path to the executable (the key in the JSON is `sprightBin`)
- run `yarn install` and `npx tsc`
- `node build/tools/script_builder/builder.js && node build/tools/spritesheet_builder/builder.js && node build/tools/map_builder/builder.js`

Now you can start the game server by running `node build/server/server.js` and the client server using `npx webpack serve`. The game can be played by navigating to http://localhost:8080/

When you make changes to the game files, you need to manually run the *_builder scripts. As such, I usually launch the server by running a command like the following:
```
node build/tools/script_builder/builder.js && node build/tools/spritesheet_builder/builder.js && node build/tools/map_builder/builder.js && node build/server/server.js
```

You can create a deployment package by running `./tools/deploy/create_packages.sh`.

## Development - with database

In order to enable game saving/persistence, you need to start a local postgres instance and edit `local_config.json` appropriately.

The easiest way to start postgres for this game is to build a deployment package and run `docker compose up db` from the `build/packages` directory. Afterwards you need to set `noDatabase` to `false` in local_config.json.

Please note that in order to run the server with a database the session server has to be running as well. It needs to be started before the game server by running `node build/server/player_session_server.js`. Afterwards the game server can be started as normal.

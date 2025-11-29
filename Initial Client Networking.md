#Minecraft's Initial Server Networking

- The server starts a TCP listener in `ServerConnectionListener#startTcpServerListener`. This is only done once in `DedicatedServer#initServer`.
	- Adds `ServerBootstrap` into the channel's collection, this bootstrap gets a `ChannelInitializer` that configures initial socket connections.

- Initial client data from a Netty socket is first processed by the set `ChannelInitializer` in `ServerBootstrap`, here the channel has its pipeline configured to decode bytes into packets, then that pipeline gets a `Connection` at *"packet_handler"*, and so on.
	- A `Connection` is created, if the server is running through a proxy, proxy related logic will run first, then the connection gets added to a queue of pending connections, finally, the connection gets a fresh `ServerHandshakePacketListenerImpl` assigned to `packetListener` in anticipation of the client returning a `ClientIntentionPacket`.
	- Every server tick, pending connections get moved to the active list of connections - this is where all server connections are ticked. If anything goes wrong, or the client is no longer marked as connected, the server disconnects the client and removes the connection.
	- Since by now the client's connection is configured, incoming packets are distributed to the `Connection` associated with this pipeline through its `Connection#channelRead0` method.

- **Handshake Stage**
	- The client now sends out a `ClientIntentionPacket` to the server, depending on the intention type, a different event is triggered.
		- LOGIN will run `ServerHandshakePacketListenerImpl#beginLogin`.
		- STATUS will return the server's current status, if replies by the server are enabled.
		- TRANSFER will first check if transfers are allowed, if not the connection is terminated, otherwise `ServerHandshakePacketListenerImpl#beginLogin` will run with a transfer flag.
	- The server sets the connection's `packetListener` set to a fresh instance of `ServerLoginPacketListenerImpl` in anticipation for receiving packets associated with a login.

- **Hello Stage**
	- The server now begins the login process, waiting for a Hello packet.
		- Note: If this whole login process takes over 600 ticks(30 seconds), the connection is closed and the client is kicked.
		- When the server receives `ServerboundHelloPacket` it runs `ServerLoginPacketListenerImpl#handleHello` which saves the given UUID and username.

- **Key Stage**
	- The server now determines if authentication is necessary, otherwise it will use the client's given `GameProfile`.
		- If so, it sets the state to be of KEY and responds with a `ClientboundHelloPacket`; this packet contains the `server ID`, `public key`, and a `challange token`
			- The server then waits for a response of `ServerboundKeyPacket` from the client, when received, it will compare the original `challenge token` to the one returned, if it's valid it sets the state to AUTHENTICATING and set's the connections encryption key to the secret it has received from the client.
			- From there the server makes a request to Mojang's authentication servers to verify the client's identity.
				- Note: Before the client sends its `ServerboundKeyPacket` to the server, it will send a POST request to Mojang's servers, including its `accessToken`, `selectedProfile`, and `serverHashId`, this is then checked by the server to validate the authenticity of the client.
			- Once the client is authenticated by the server through Mojang's servers, the state is set to VERIFYING, then on the next server tick the server runs `ServerLoginPacketListenerImpl#verifyLoginAndFinishConnectionSetup`.
		- If not, it will check if the server is using a proxy use proxy logic, otherwise it will change the state to VERIFYING, and use the client's sent `GameProfile`.

- **Compression/Authentication Stage**(maybe)
	- Here, if the server has compression enabled, a `ClientboundLoginCompressionPacket` is sent to the client so that it knows what level of compression it will be dealing with.
	- Then, the server checks for players that match the existing `GameProfile`(logged-in) on the server.
		- If they exist, the state is set to WAITING_FOR_DUPE_DISCONNECT, then on the following tick, the server checks for pre-existing players with the same `GameProfile`, this will continue until there are none, or if the client times out.

	- The state is then set to PROTOCOL_SWITCHING, sending the client a `ClientboundLoginFinishedPacket` and adds the player's profile to paper's `filledProfileCache`.
	- The server then expects the client to send a `ServerboundLoginAcknowlegedPacket`, once received, it will set the `packetListener` to a new instance of `ServerConfigurationPacketListenerImpl` and setting state to ACCEPTED.
	 - Breakdown:
		- `ServerboundHelloPacket`(Client -> Server): The client's `username` and `UUID`
		- If in online-mode:
			- `ClientboundHelloPacket`(Server -> Client): The `server ID`, the server's `public key`, a random `challange token`, and a `flag` telling the client if it should authenticate through Mojang servers.
			- `ServerboundKeyPacket`(Client -> Server): The original `challange token`, encrypted with the server's public key and a `shared secret`, encrypted the same way. 

		- `ClientboundLoginCompressionPacket`(Server -> Client): The servers compression level so that the client can react accordingly to incoming packets.
		- `ClientboundLoginFinishedPacket`(Server -> Client): The final, authenticated `GameProfile` containing the client's final UUID, username, and a `PropertyMap` containing skin properties.
		- `ServerboundLoginAcknowledgedPacket`(Client -> Server):

- **Configuration Stage**
	- The server now goes through configuring the client
		- The server first sends out a `ClientboundCustomPayloadPacket` that contains the server's brand, then if the server has set server links, it'll send out `ClientboundServerLinksPacket`, then the `ClientboundUpdateEnabledFeaturesPacket` is sent.
		- The server then begins constructing a queue of configuration tasks. This queue includes:
			- `SynchronizeRegistriesTask`:
			- If code of conduct is not empty:
				- `ServerCodeOfConductConfigurationTask`: A task that will send out a `ClientboundCodeOfConductPacket`. This is just a string.

			- If the server has a resource pack:
				- `ServerResourcePackConfigurationTask`: A task that will send out the `ClientboundResourcePackPushPacket`. Which is made up of a pack ID, URL, hash, a required boolean, and if the server should prompt the client.

			- `PaperConfigurationTask`: This simply calls the `AsyncPlayerConnectionConfigureEvent` in paper's API if any listeners are registered.
			- `PrepareSpawnTask`:
				- This FIRST task attempts to load the player's data, if none exists, newPlayer is set to true. If the player has data, use it, otherwise use default values.
				- Attempt to get the world the player's world, then attempt to get the player's position, then calculate the player's spawn world and position.
				- There is a lot more logic performed, but I'm ignoring that.

			- `JoinWorldTask`: An empty task that contains a `ClientboundFinishConfigurationPacket` to notify the client that configuration has ended.

		- The client sends these possible packets to the server:
			- If the server sent out a code of conduct the client must return:
				- `ServerboundAcceptCodeOfConductPacket`: Empty packet that serves as an acceptance of the code of conduct.

			- If the server has a set resource pack the client must return:
				- `ServerboundResourcePackPacket`: Returned with a status of one of:
					- `SUCCESSFULLY_LOADED`, `DECLINED`, `FAILED_DOWNLOAD`, `ACCEPTED`, `DOWNLOADED`, `INVALID_URL`, `FAILED_RELOAD`, `DISCARDED`

			- If the server supports pack selection, the client may send:
				- `ServerboundSelectKnownPacks`:

			- `ServerboundFinishConfigurationPacket`: This is the only universally required packet response from the client to the server.

- Play State
	- The server then runs `PrepareSpawnTask#spawnPlayer` once it receives `ServerboundFinishConfigurationPacket` from the client. If `PrepareSpawnTask` is ready, a `ServerPlayer` is constructed in `PrepareSpawnTask.Ready#spawn`, this method later calls `PlayerList#placeNewPlayer` to bring the player into the world.

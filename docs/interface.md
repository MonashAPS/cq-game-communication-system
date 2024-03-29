# Interface

The overall cycle of communications between the server and clients looks like this:

1. The server should wait for an initial message containing a list of connected clients. This message will mean all the
clients have connected (or have timed out) and that the game should start now. This message will look like this:

```json
{"clients": [{"id": "client1", "name": "Client One!"}, {"id": "client2", "name": "Client Two!"}]}
```

2. After the server has read the clients' information, it will send some initialization messages to clients. These
messages are supposed to send some basic info about the game (as opposed to that turn) to clients so they can do
some preprocessing if they need. An example of what the game server could send here is the map of the game. The format
of these messages (like all other messages) is an object where the key defines the receiver of the message and the value
defines the message itself. If the message should go to everyone, use an empty string as the key.

After each message is sent, you should also print a floating point number, denoting how much time the clients
are given to process that message before the new message comes. Time should be in seconds.
```json
{"": {"map_objects": [{"type": "wall", "location": [1, 2]}, {"type": "wall", "location": [2, 2]}]}}
0.5
```

3. Once all init messages are sent, the server should send a message `"END_INIT"` so the clients would know the game
cycle is started now. The keyword `"END_INIT"` is defined in `src/config.py`. You do not need to print a time after the
end init message.
```json
"END_INIT"
```

4. Now that the game server has read the clients' information and has sent the initializing messages, it can start the
main cycle. The main cycle is basically a number of back and forth messages between the server and the clients where the 
server sends a message and awaits the clients' response. The server can send different messages to different clients 
or send the same message to all clients (exactly like the init messages). In order to send different messages to
different clients, it should print:
```json
{"client1": "Message for Client 1", "client2": "Message for Client 2"}
```

where the key is the client's **id** from the initial info. If you want to send the same message to all clients then
pass the key as empty string:
```json
{"": "Same message for all clients."}
```

5. The above message would be the first part of the repeated cycle. After that message, the server should specify how
long it would like to wait for the clients to respond. The server has to specify this number every turn and it does not
need to be the same number. The unit of this wait time is seconds and the given number can be smaller than 1. If the
server passes a number <= 0 the game will crash.
6. Once these two messages have been printed by the server, each client will receive a message like this:

```json
{"message": "Message for this client", "time": 0.5}
```

This tells the client that they have 0.5 seconds to respond to the sent message.

6. Now it's the clients' turn to respond. Each client should print exactly one line in their standard output. If they
print their response too late (after the specified timeout time) the message will be discarded and the server will
receive `None` as the client's message.
7. Once the clients have responded, the same cycle repeats again from step 2 until the server says `"END"`.


## Example

Here is an example of the messages sent and received by the server and one client in the course of one game. `>` means
that line is received and `<` means that line is a message sent out. The first line received by the server is sent
by the GCS and that's the only message that GCS injects in, in the whole process.

**Server**
```shell
# Initial message sent by GCS
> {"clients": [{"id": "client_id", "name": "Client Name!", "image": "client_image:latest"}]}
# The first initialize-world message sent by the server to all clients
< {"": {"map": [[1,1,1],[1,0,1],[1,1,1]]}}
# They are given 0.1 seconds
< 0.1
# The second initialize-world message sent by the server to all clients
< {"": {"tanks": [{"name": "tank one"}, {"name": "tank two"}]}}
# They are given 0.1 seconds
< 0.1
# The third initialize-world message sent by the server to clients, with different values
< {"client-1": {"your-tank-id": "tank-1"}, "client-2": "your-tank-id": "tank-2"}
# They are given half a second to process everything
< 0.5
# The end of the initialize-world sequence
< "END_INIT"
# Send a message to clients for the first turn
< {"client-1": "A message for this client"}
# Define the turn timeout
< 10

# Now wait for the clients...

# Clients have responded
> {"client-1": "Message Received!"}
# Send a message for the next turn and this time send it to all clients
< {"": "Message for all clients!"}
# This time the clients have half a second
< 0.5

# Wait for the clients...

# Time is up. The client we are examining didn't say anything.
> {"client-1": null}
# Finish the game
< "END"
```

**Client One**
```shell
# The first initializing message received:
> {"message": {"map": [[1,1,1],[1,0,1],[1,1,1]]}, "time": 0.1}
# The second initialize-world message received:
> {"message": {"tanks": [{"name": "tank one"}, {"name": "tank two"}]}, "time": 0.1}
# The third initialize-world message received:
> {"message": {"your-tank-id": "tank-1"}, "time": 0.5} 
# The signal for the end of the init messages is received
> "END_INIT"

# First turn's message received
> {"message": "A message for this client", "time": 10}
# Respond
< "Message Received!"

# Now wait for the next message from the server...

# Next turn
> {"message": "Message for all clients!", "time": 0.5}
# Don't Respond

# No further messages are received as the server finished the game.
```

## Notes

- All messages should be in JSON. So even if the server/client wants to send a single string, it
should be quoted like `"Simple String"` because `"Simple String"` is a JSON message but `Simple String` isn't. With the
same logic, if you want to send a number, you shouldn't quote it in "" because `"20"` won't be parsed as a number but
`20` will. So it's best to write your programs to always apply a JSON encoding write before it prints the message out.

- "One message" in this context means one line of standard output. So make sure your JSON messages
are printed in a single line not across multiple lines otherwise GCS has to assume they are separate messages.
Be mindful that you can't print more than 8100 characters in one line otherwise the message will be broken.

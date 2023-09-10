---
external: false
title: "[Rust] Actix-Web Actorless Websocket Server Part 2"
description: "A tutorial about writing an actorless websocket server in rust."
date: 2023-09-10
---
This is part 2 of my tutorial on creating an Actix-Web Actorless Websocket Server (you can 
see the previous part [here](/blog/actix-websocket-actorless1)). 
In the last tutorial, we got a healthcheck running and created our web socket 
server struct. In this session, I want to cover sessions (pun not intended), specifically 
user sessions and by the end of this part you hopefully could have a running
web socket server that you can test out.

Alright, so in the last part, we were left with this piece of code.

```rust
// main.rs
use actix_web::{HttpServer, App, web, Responder};
use std::collections::HashMap;
use tokio::sync::mpsc;

type SessionStore = HashMap<i32, mpsc::UnboundedSender<String>>;
enum WebSocketCommands {
    UserConnect { user_id: i32, user_sender: mpsc::UnboundedSender<String> },
    UserMessage { user_id: i32, user_message: String }
}

struct WebSocketServer {
    session_store: SessionStore,
    ws_server_receiver: mpsc::UnboundedReceiver<WebSocketCommands>
}

impl WebSocketServer {

    fn new() -> (Self, UserSessionFactory) {
        let (ws_server_sender, ws_server_receiver) = mpsc::unbounded_channel::<WebSocketCommands>();
        let server = Self {
            session_store: SessionStore::new(),
            ws_server_receiver
        };
        let session_factory = UserSessionFactory {
            ws_server_sender
        };
        (server, session_factory)
    }

    async fn user_connect(
        &mut self,
        user_id: i32,
        user_sender: mpsc::UnboundedSender<String>
    ) {
        self.session_store.insert(user_id, user_sender);
    }

    async fn user_sends_message(
        &mut self,
        user_id: i32,
        message: String
    ) {
        for (_, user_sender) in self.session_store.iter() {
            user_sender.send(format!("[UID: {}]: {}", user_id, message.clone())).unwrap();
        }
    }

    async fn run(mut self) -> std::io::Result<()> {
        use WebSocketCommands::*;
        while let Some(msg) = self.ws_server_receiver.recv().await {
            match msg {
                UserConnect { user_id, user_sender } => self.user_connect(user_id, user_sender).await,
                UserMessage { user_id, user_message } => self.user_sends_message(user_id, user_message).await
            }
        }
        Ok(())
    }
}


async fn healthcheck() -> impl Responder {
    "rust-websocket-tutorial is live"
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    env_logger::init_from_env(env_logger::Env::new().default_filter_or("info"));
    let port = 5000;

    log::info!("rust-websocket-tutorial will be running on http://0.0.0.0:{port}");
    let http_server = HttpServer::new(move || {
        App::new()
            .service(web::resource("/healthcheck").route(web::get().to(healthcheck)))
    })
    .workers(2)
    .bind(("0.0.0.0", port))?
    .run();

    http_server.await?;

    Ok(())
}
```

But nothing's running because we haven't defined our `UserSessionFactory`.
Remember from the previous article that the `UserSessionFactory` creates (as it's name suggests) user sessions and keeps an instance of the `mpsc::UnboundedSender` type so 
that user sessions can pass messages to the web socket server. The code is as follows:
```rust
#[derive(Clone)]
struct UserSessionFactory {
    ws_server_sender: mpsc::UnboundedSender<WebSocketCommands>
}

impl UserSessionFactory {
    fn create_session(
        &self, 
        user_id: i32, 
        session: actix_ws::Session
    ) -> UserSession {
        let ws_server_sender = self.ws_server_sender.clone();
        UserSession { user_id, ws_server_sender, session }
    }
}
```
Pretty simple right? It just has one method which creates user sessions and passes it the
`mpsc::UnboundedSender` type which user sessions use to communicate with the web socket server. *Nice!*

Well now, there's only one more piece of the puzzle left! Our user sessions! Our user session, is going to need the `ws_server_sender` attribute from the UserSessionFactory
and it also needs a way to differentiate itself from other sessions, here I am just using
the user id. Then it also has this `actix_ws::Session` attribute. What is this? Welp,
to put it simply, it's the attribute that the user session uses to actually send messages
to the client that is establishing the websocket connection. Here's the code for the struct.

```rust
struct UserSession {
    user_id: i32,
    ws_server_sender: mpsc::UnboundedSender<WebSocketCommands>,
    session: actix_ws::Session
}
```

Alright, so our `UserSession` is going to need to be able to handle messages from two
different sources. The first source is the server, and to make things simple I'm just going to send messages from the server back straight to the user, here we are using the `actix_ws::Session` type that we mentioned earlier. Here's the code for that.
```rust
impl UserSession {
    async fn handle_server_message(&mut self, msg: Option<String>) {
        let Some(msg) = msg else { return; };
        self.session.text(msg).await.unwrap();
    }
}
```
Our second source is the user themselves. So, to handle that we're simply going to forward
any text we receive straight to the web socket server and here it is.
```rust
impl UserSession {
    async fn handle_session_message(&self, msg: Option<Result<Message, ProtocolError>>) -> Option<CloseReason> {
        use Message::*;
        use WebSocketCommands::*;
        let Some(msg) = msg else { return None; };
        let Ok(msg) = msg else { return None; };
        match msg {
            Text(text) => {
                self.ws_server_sender.send(UserMessage { user_id: self.user_id, user_message: text.to_string() }).unwrap();
                return None;
            },
            Close(reason) => return reason,
            _ => return None
        };
    }
    // ...
}
```
Here, we're using the `UserMessage` variant from the `WebSocketCommands` enum to
send this message to the server. Then, similar to our web socket server we are going to have a run loop, which waits for a message and executes the corresponding code to handle that message.
```rust
enum UserSessionMessage {
    SessionMessage(Option<Result<Message, ProtocolError>>),
    ServerMessage(Option<String>)
}

impl UserSession {
    // ...
    async fn run(mut self, mut msg_stream: MessageStream) {
        use WebSocketCommands::*;
        use UserSessionMessage::*;
        let (session_sender, mut session_receiver) = mpsc::unbounded_channel::<String>();
        let user_sender = session_sender;
        let user_id = self.user_id;
        self.ws_server_sender.send(UserConnect { user_id, user_sender }).unwrap();

        let reason = loop {

            let server_stream = session_receiver.recv().into_stream().map(ServerMessage);
            let session_stream = msg_stream.recv().into_stream().map(SessionMessage);
            let mut stream = (server_stream, session_stream).merge();
            let Some(msg) = stream.next().await else { continue; };
            match msg {
                SessionMessage(msg) => { 
                    if let Some(reason) = self.handle_session_message(msg).await { 
                        break reason;
                    };
                },
                ServerMessage(msg) => self.handle_server_message(msg).await
            };

        };
        self.session.close(Some(reason)).await.unwrap();
    }
}
```
So, our run function creates a channel, then it passes the sender/producer side of that 
channel to the web socket server using the `UserConnect` message. 
This will then be handled by our web socket server here.
```rust
    async fn run(mut self) -> std::io::Result<()> {
        use WebSocketCommands::*;
        while let Some(msg) = self.ws_server_receiver.recv().await {
            match msg {
                UserConnect { user_id, user_sender } => self.user_connect(user_id, user_sender).await,
                UserMessage { user_id, user_message } => self.user_sends_message(user_id, user_message).await
            }
        }
        Ok(())
    }
```
Which is then handled by the `user_connect` method. Going back to the run function
in our `UserSession`, I then used the `MergeStreams` crate and the `futures-util` crate
to simplify things and *merge* these two streams together. This makes it so that,
I can treat both messages from the server and messages from the client as one source.
Which I can then handle separately in our match function.

I've been trying to make the run loop easier to read so you can see everything 
that is going on here. Compare this to the example in Actix Web's Example Github Repo.
```rust
// actix chat actorless github example.

let close_reason = loop {
        // most of the futures we process need to be stack-pinned to work with select()

        let tick = interval.tick();
        pin!(tick);

        let msg_rx = conn_rx.recv();
        pin!(msg_rx);

        // TODO: nested select is pretty gross for readability on the match
        let messages = select(msg_stream.next(), msg_rx);
        pin!(messages);

        match select(messages, tick).await {
            // commands & messages received from client
            Either::Left((Either::Left((Some(Ok(msg)), _)), _)) => {
                // ...
            }

            // client WebSocket stream error
            Either::Left((Either::Left((Some(Err(err)), _)), _)) => {
                // ...
            }

            // client WebSocket stream ended
            Either::Left((Either::Left((None, _)), _)) => break None,

            // chat messages received from other room participants
            Either::Left((Either::Right((Some(chat_msg), _)), _)) => {
                // ...
            }

            // all connection's message senders were dropped
            Either::Left((Either::Right((None, _)), _)) => unreachable!(
                "all connection message senders were dropped; chat server may have panicked"
            ),

            // heartbeat internal tick
            Either::Right((_inst, _)) => {
                // if no heartbeat ping/pong received recently, close the connection
                // ...
            }
        };
    };

```
In my opinion this loop handles too much logic making it too hard to grasp the over arching
idea of what's going in the loop. In the implementation that I wrote, I combined the two
sources of information that we talked about previously (the server and the user) into one 
source, then handling messages from each source separately. As compared to the actix 
web example that uses a select statement instead. Funny that it even states in the code
that the select statement is gross and I agree!
```rust
// TODO: nested select is pretty gross for readability on the match
```

Now, we just need to run the server in our main function and give it an endpoint to live on.

```rust
pub fn get_uid_from_header(req: HttpRequest) -> Option<i32> {
    let uid = req
        .headers()
        .get("uid")
        .map(|v| v.to_str().ok())
        .flatten()
        .map(|s| s.to_string())
        .map(|s| s.parse::<i32>().ok())
        .flatten();
    uid
}

async fn chat_ws(
    req: HttpRequest,
    stream: web::Payload,
    session_factory: web::Data<UserSessionFactory>
) -> Result<HttpResponse, Error> {
    let (res, session, msg_stream) = actix_ws::handle(&req, stream)?;
    let Some(uid) = get_uid_from_header(req) else {
        session.close(Some(CloseReason { code: actix_ws::CloseCode::Error, description: Some("uid is missing".to_string()) })).await.unwrap();
        return Ok(res);
    };
    let session = session_factory.create_session(uid, session);
    spawn_local(session.run(msg_stream));

    Ok(res)
}

#[actix_web::main]
async fn main() -> std::io::Result<()>{
    env_logger::init_from_env(env_logger::Env::new().default_filter_or("info"));

    let (websocket_server, session_factory) = WebSocketServer::new();

    let websocket_server = spawn(websocket_server.run());
    let port = 5000;

    log::info!("Server will be live on http://0.0.0.0:{port}");
    let http_server = HttpServer::new(move || {
        App::new()
            .app_data(web::Data::new(session_factory.clone()))
            .service(web::resource("/healthcheck").route(web::get().to(healthcheck)))
            .service(web::resource("/ws").route(web::get().to(chat_ws)))
    })
    .workers(2)
    .bind(("0.0.0.0", port))?
    .run();

    try_join!(http_server, async move { websocket_server.await.unwrap() })?;
    Ok(())
}
```
Alright, let's look at the things we have here. In our main function, we added
these two lines.
```rust
    // ...
    let (websocket_server, session_factory) = WebSocketServer::new();
    let websocket_server = spawn(websocket_server.run());
    // ...
```
This creates our web socket server and spawns a tokio task (read more about tokio tasks [here](https://docs.rs/tokio/latest/tokio/task/index.html)). Another part of the main function, that's probably good to talk about is this line.
```rust
    try_join!(http_server, async move { websocket_server.await.unwrap() })?;
```
This line just waits for both the `http_server` and `websocket_server` to finish execution 
(which it never does, because of the loop we had in the `websocket_server`).
We also have the code for our handler.
```rust
async fn chat_ws(
    req: HttpRequest,
    stream: web::Payload,
    session_factory: web::Data<UserSessionFactory>
) -> Result<HttpResponse, Error> {
    let (res, session, msg_stream) = actix_ws::handle(&req, stream)?;
    let Some(uid) = get_uid_from_header(req) else {
        session.close(Some(CloseReason { code: actix_ws::CloseCode::Error, description: Some("uid is missing".to_string()) })).await.unwrap();
        return Ok(res);
    };
    let session = session_factory.create_session(uid, session);
    spawn_local(session.run(msg_stream));

    Ok(res)
}
```
All that the handler does is establish a web socket connection with the client,
obtains a user id from the header (to identity each session) and
then creates a session.

Finally, to run our web socket server, we can just do `cargo run`.
To test the websocket server out, I used postman and created two websocket connections. 
Also, don't forget to include the user id in the header.

![postman-uid1](/images/actix-websocket-actorless/postman-uid1.png)
![postman-uid2](/images/actix-websocket-actorless/postman-uid2.png)

You can find the code for this tutorial [here](https://github.com/BrynGhiffar/rust-websocket-tutorial).
That's it for this tutorial and thank you for reading!
# Real-time reference — socket.io (midu-style)

From `ChatWebApplication` (`socket.io` ^4.8.3 server + `socket.io-client` ^4.8.3 client).
This is the canonical real-time implementation; `ChatApp` is an earlier, buggier draft.

## Naming conventions
- **Chat/room events are `snake_case`:** `setup`, `join_room`, `new_message`, `conversation_updated`
- **WebRTC call events are `kebab-case`:** `call-user`, `incoming-call`, `answer-call`,
  `call-answered`, `ice-candidate`, `reject-call`, `call-rejected`, `end-call`, `call-ended`
- Payload keys stay `camelCase`.
- **Rooms:** every user joins a personal room named by their raw Mongo `userId`.
  Every conversation is a room named by the conversation `_id`.

---

## Client — `src/lib/socket.js`
A single shared instance, default-exported. No context, no provider.
```js
import { io } from 'socket.io-client'

const socket = io('https://chatwebapplication-vum6.onrender.com', { withCredentials: true })

export default socket
```

## Client — consuming in `useEffect`, always with `socket.off` cleanup
```jsx
// components/MessageArea.jsx
useEffect(() => {
  if (!activeConv?._id) return

  socket.emit('join_room', activeConv._id)

  const handleNew = (list) => setMessages(list)
  socket.on('new_message', handleNew)

  return () => socket.off('new_message', handleNew)
}, [activeConv?._id])
```
```jsx
// components/ConversationList.jsx — personal room + RTK Query refetch
useEffect(() => {
  if (!myId) return

  socket.emit('setup', myId)

  const handleUpdate = () => refetch()
  socket.on('conversation_updated', handleUpdate)

  return () => socket.off('conversation_updated', handleUpdate)
}, [myId, refetch])
```
Note the pattern: **the socket event does not carry the new data — it just triggers an RTK
Query `refetch()`.** Only `new_message` pushes a payload (the whole message list).

---

## Server — `index.js`, with `global.io` so controllers can emit
```js
const httpServer = require('http').createServer(app)
const io = require('socket.io')(httpServer, { cors: { origin: allowedOrigins, credentials: true } })

global.io = io

io.on('connection', (socket) => {
  // ====== Join personal room (named by userId)
  socket.on('setup', (userId) => { if (userId) socket.join(userId) })

  // ====== Join a conversation room
  socket.on('join_room', (convId) => { socket.join(convId) })

  // ====== WebRTC call signaling (relayed via each user's personal room)
  socket.on('call-user', ({ to, from, caller, callType, offer }) => {
    io.to(to).emit('incoming-call', { from, caller, callType, offer })
  })
  socket.on('answer-call', ({ to, answer }) => { io.to(to).emit('call-answered', { answer }) })
  socket.on('ice-candidate', ({ to, candidate }) => { io.to(to).emit('ice-candidate', { candidate }) })
  socket.on('reject-call', ({ to }) => { io.to(to).emit('call-rejected') })
  socket.on('end-call', ({ to }) => { io.to(to).emit('call-ended') })
})

httpServer.listen(port, () => { console.log(`server is running`) })
```
Important: with sockets you must `httpServer.listen(...)`, **not** `app.listen(...)`
(the `ChatApp` repo gets this wrong — don't copy that).

## Server — emitting from a controller via `global.io`
```js
const sendMessage = async (req, res) => {
  try {
    const { contentType = 'text', content, conversation } = req.body

    const isExistConv = await conversationSchema.findOne({ _id: conversation })
    if (!isExistConv) return res.status(400).send({ message: 'Conversation not found' })

    const message = new messageSchema({ contentType, content, conversation, sender: req?.user?._id })
    await message.save()

    isExistConv.lastMessage = content
    await isExistConv.save()

    const messageList = await messageSchema.find({ conversation })

    // ====== Push the new list to the conversation room
    global.io.to(conversation).emit('new_message', messageList)
    // ====== Nudge both participants' personal rooms to refetch their sidebar
    global.io.to(String(isExistConv.creator)).to(String(isExistConv.participant)).emit('conversation_updated')

    res.status(200).send('sent')
  } catch (error) { console.log(error) }
}
```
`String(...)` around ObjectIds before `.to()` — room names must be strings.

## Message schema — media messages are just a String + a type enum
```js
contentType: { type: String, default: 'text', enum: ['text', 'image', 'voice', 'video'], required: true },
content:     { type: String, required: true },
sender:      { type: mongoose.Types.ObjectId, ref: 'user', required: true },
conversation:{ type: mongoose.Types.ObjectId, ref: 'convSchema', required: true },
```

## Endpoint paths in the chat app (lowercase, no separators)
`/auth/signup`, `/auth/signin`, `/auth/getprofile`, `/convo/addnewfriend`, `/convo/list`,
`/convo/sendmessage`, `/convo/getmessage/:conversation`

## Auth for sockets
Cookie-based, piggybacking on the HTTP session: client passes `withCredentials: true`,
server CORS sets `credentials: true`. There is **no** socket-level JWT handshake middleware.

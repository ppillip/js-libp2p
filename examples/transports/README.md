# [Transports](http://libp2p.io/implementations/#transports)

libp2p doesn't make assumptions for you, instead, it enables you as the developer of the application to pick the modules you need to run your application, which can vary depending on the runtime you are executing.

`libp2p는 가정을하지 않으며 대신 응용 프로그램 개발자가 응용 프로그램을 실행하는 데 필요한 모듈을 선택하게합니다. 응용 프로그램은 실행중인 런타임에 따라 달라질 수 있습니다` 

A libp2p node can use one or more Transports to dial and listen for Connections. 

`libp2p 노드는 하나 이상의 전송을 사용하여 전화 접속 연결을 수신 대기 할 수 있습니다.` 

These transports are modules that offer a clean interface for dialing and listening, defined by the [interface-transport](https://github.com/libp2p/interface-transport) specification. 

[interface-transport](https://github.com/libp2p/interface-transport) `에 정의된, 전송 모듈은 dialing 및 listening을 위한 깔끔한 인터페이스를 제공합니다.` 

Some examples of possible transports are: TCP, UTP, WebRTC, QUIC, HTTP, Pigeon and so on. 

`가능한 전송의 예로는 TCP, UTP, WebRTC, QUIC, HTTP, Pigeon 등이 있습니다.`

A more complete definition of what is a transport can be found on the [interface-transport](https://github.com/libp2p/interface-transport) specification. 

`전송 장치에 대한보다 상세한 정의는` [interface-transport](https://github.com/libp2p/interface-transport) `스펙에서 찾을 수 있습니다.` 

A way to recognize a candidate transport is through the badge 

`후보 운송 수단을 인식하는 방법은 배지를 통한 것입니다.`:

[![](https://raw.githubusercontent.com/diasdavid/interface-transport/master/img/badge.png)](https://raw.githubusercontent.com/diasdavid/interface-transport/master/img/badge.png)

## 1. TCP로 libp2p 번들 생성하기

`libp2p를 사용하여, 자신만의 libp2p 번들을 만들고 싶을때, 모듈 집합을 선택하고 원하는 네트워크 속성으로 필요한 스텍을 만듭니다.`

`이 예제에서는 TCP를 사용하여 번들을 생성합니다. 전체 소스는` [1.js](./ 1js) `파일 에서 찾을 수 있습니다.`

`총 5개의 deps가 필요합니다. 다음과 같이 설치 하세요 : `

```bash
> npm install libp2p libp2p-tcp peer-info async @nodeutils/defaults-deep
```

`그런 다음, 좋아하는 텍스트 편집기에서` `.js` `확장자로 파일을 생성하십시오. 나는 ` `1.js` `라고 붙였습니다`

`우선 우리 자신의 번들을 만드는 것입니다!` 삽입 :

```JavaScript
'use strict'

const libp2p = require('libp2p')
const TCP = require('libp2p-tcp')
const PeerInfo = require('peer-info')
const waterfall = require('async/waterfall')
const defaultsDeep = require('@nodeutils/defaults-deep')

// This MyBundle class is your libp2p bundle packed with TCP
class MyBundle extends libp2p {
  constructor (_options) {
    const defaults = {
      // modules is a JS object that will describe the components
      // we want for our libp2p bundle
      modules: {
        transport: [
          TCP
        ]
      }
    }

    super(defaultsDeep(_options, defaults))
  }
}
```

`이제 libp2p를 확장하여 MyBundle 클래스를 만들었습니다. 클래스를 사용하여 노드를 만들어 봅시다.`

`앞으로 async / waterfall 을 코드 구조로 사용 하겠지만, 꼭 그럴필요는 없습니다. 동일한 파일에 추가하세요 :`

```JavaScript
let node

waterfall([
  // First we create a PeerInfo object, which will pack the
  // info about our peer. Creating a PeerInfo is an async
  // operation because we use the WebCrypto API
  // (yeei Universal JS)
  (cb) => PeerInfo.create(cb),
  (peerInfo, cb) => {
    // To signall the addresses we want to be available, we use
    // the multiaddr format, a self describable address
    peerInfo.multiaddrs.add('/ip4/0.0.0.0/tcp/0')
    // Now we can create a node with that PeerInfo object
    node = new MyBundle({ peerInfo: peerInfo })
    // Last, we start the node!
    node.start(cb)
  }
], (err) => {
  if (err) { throw err }

  // At this point the node has started
  console.log('node has started (true/false):', node.isStarted())
  // And we can print the now listening addresses.
  // If you are familiar with TCP, you might have noticed
  // that we specified the node to listen in 0.0.0.0 and port
  // 0, which means "listen in any network interface and pick
  // a port for me
  console.log('listening on:')
  node.peerInfo.multiaddrs.forEach((ma) => console.log(ma.toString()))
})
```

이 작업을 실행하면 다음과 같은 결과가 발생합니다 : 

```bash
> node 1.js
node has started (true/false): true
listening on:
/ip4/127.0.0.1/tcp/61329/ipfs/QmW2cKTakTYqbQkUzBTEGXgWYFj1YEPeUndE1YWs6CBzDQ
/ip4/192.168.2.156/tcp/61329/ipfs/QmW2cKTakTYqbQkUzBTEGXgWYFj1YEPeUndE1YWs6CBzDQ
```

That `QmW2cKTakTYqbQkUzBTEGXgWYFj1YEPeUndE1YWs6CBzDQ` is the PeerId that was created during the PeerInfo generation.

## 2. Dialing from one node to another node

Now that we have our bundle, let's create two nodes and make them dial to each other! You can find the complete solution at [2.js](./2.js).

For this step, we will need one more dependency.

```bash
> npm install pull-stream
```

And we also need to import the module on our .js file:

```js
const pull = require('pull-stream')
```

We are going to reuse the MyBundle class from step 1, but this time to make things simpler, we will create two functions, one to create nodes and another to print the addrs to avoid duplicating code.

```JavaScript
function createNode (callback) {
  let node

  waterfall([
    (cb) => PeerInfo.create(cb),
    (peerInfo, cb) => {
      peerInfo.multiaddrs.add('/ip4/0.0.0.0/tcp/0')
      node = new MyBundle({ peerInfo: peerInfo })
      node.start(cb)
    }
  ], (err) => callback(err, node))
}

function printAddrs (node, number) {
  console.log('node %s is listening on:', number)
  node.peerInfo.multiaddrs.forEach((ma) => console.log(ma.toString()))
}

```

Now we are going to use `async/parallel` to create two nodes, print their addresses and dial from one node to the other. We already added `async` as a dependency, but still need to import `async/parallel`:

```js
const parallel = require('async/parallel')
```

Then,

```js
parallel([
  (cb) => createNode(cb),
  (cb) => createNode(cb)
], (err, nodes) => {
  if (err) { throw err }

  const node1 = nodes[0]
  const node2 = nodes[1]

  printAddrs(node1, '1')
  printAddrs(node2, '2')

  node2.handle('/print', (protocol, conn) => {
    pull(
      conn,
      pull.map((v) => v.toString()),
      pull.log()
    )
  })

  node1.dialProtocol(node2.peerInfo, '/print', (err, conn) => {
    if (err) { throw err }

    pull(pull.values(['Hello', ' ', 'p2p', ' ', 'world', '!']), conn)
  })
})
```

The result should be look like:

```bash
> node 2.js
node 1 is listening on:
/ip4/127.0.0.1/tcp/62279/ipfs/QmeM4wNWv1uci7UJjUXZYfvcy9uqAbw7G9icuxdqy88Mj9
/ip4/192.168.2.156/tcp/62279/ipfs/QmeM4wNWv1uci7UJjUXZYfvcy9uqAbw7G9icuxdqy88Mj9
node 2 is listening on:
/ip4/127.0.0.1/tcp/62278/ipfs/QmWp58xJgzbouNJcyiNNTpZuqQCJU8jf6ixc7TZT9xEZhV
/ip4/192.168.2.156/tcp/62278/ipfs/QmWp58xJgzbouNJcyiNNTpZuqQCJU8jf6ixc7TZT9xEZhV
Hello p2p world!
```

## 3. Using multiple transports

Next, we want to be available in multiple transports to increase our chances of having common transports in the network. A simple scenario, a node running in the browser only has access to HTTP, WebSockets and WebRTC since the browser doesn't let you open any other kind of transport, for this node to dial to some other node, that other node needs to share a common transport.

What we are going to do in this step is to create 3 nodes, one with TCP, another with TCP+WebSockets and another one with just WebSockets. The full solution can be found on [3.js](./3.js).

In this example, we will need to also install `libp2p-websockets`, go ahead and install:

```bash
> npm install libp2p-websockets
```

We want to create 3 nodes, one with TCP, one with TCP+WebSockets and one with just WebSockets. We need to update our `MyBundle` class to contemplate WebSockets as well:

```JavaScript
const WebSockets = require('libp2p-websockets')
// ...

class MyBundle extends libp2p {
  constructor (_options) {
    const defaults = {
      modules: {
        transport: [
          TCP,
          WebSockets
        ]
      }
    }

    super(defaultsDeep(_options, defaults))
  }
}
```

Now that we have our bundle ready, let's upgrade our createNode function to enable us to pick the addrs in which a node will start a listener.

```JavaScript
function createNode (addrs, callback) {
  if (!Array.isArray(addrs)) {
    addrs = [addrs]
  }

  let node

  waterfall([
    (cb) => PeerInfo.create(cb),
    (peerInfo, cb) => {
      addrs.forEach((addr) => peerInfo.multiaddrs.add(addr))
      node = new MyBundle({ peerInfo: peerInfo })
      node.start(cb)
    }
  ], (err) => callback(err, node))
}
```

As a rule, a libp2p node will only be capable of using a transport if: a) it has the module for it and b) it was given a multiaddr to listen on. The only exception to this rule is WebSockets in the browser, where a node can dial out, but unfortunately cannot open a socket.

Let's update our flow to create nodes and see how they behave when dialing to each other:

```JavaScript
parallel([
  (cb) => createNode('/ip4/0.0.0.0/tcp/0', cb),
  // Here we add an extra multiaddr that has a /ws at the end, this means that we want
  // to create a TCP socket, but mount it as WebSockets instead.
  (cb) => createNode(['/ip4/0.0.0.0/tcp/0', '/ip4/127.0.0.1/tcp/10000/ws'], cb),
  (cb) => createNode('/ip4/127.0.0.1/tcp/20000/ws', cb)
], (err, nodes) => {
  if (err) { throw err }

  const node1 = nodes[0]
  const node2 = nodes[1]
  const node3 = nodes[2]

  printAddrs(node1, '1')
  printAddrs(node2, '2')
  printAddrs(node3, '3')

  node1.handle('/print', print)
  node2.handle('/print', print)
  node3.handle('/print', print)

  // node 1 (TCP) dials to node 2 (TCP+WebSockets)
  node1.dialProtocol(node2.peerInfo, '/print', (err, conn) => {
    if (err) { throw err }

    pull(pull.values(['node 1 dialed to node 2 successfully']), conn)
  })

  // node 2 (TCP+WebSockets) dials to node 2 (WebSockets)
  node2.dialProtocol(node3.peerInfo, '/print', (err, conn) => {
    if (err) { throw err }

    pull(pull.values(['node 2 dialed to node 3 successfully']), conn)
  })

  // node 3 (WebSockets) attempts to dial to node 1 (TCP)
  node3.dialProtocol(node1.peerInfo, '/print', (err, conn) => {
    if (err) {
      console.log('node 3 failed to dial to node 1 with:', err.message)
    }
  })
})
```

`print` is a function created using the code from 2.js, but factored into its own function to save lines, here it is:

```JavaScript
function print (protocol, conn) {
  pull(
    conn,
    pull.map((v) => v.toString()),
    pull.log()
  )
}
```

If everything was set correctly, you now should see the following after you run the script:

```Bash
> node 3.js
node 1 is listening on:
/ip4/127.0.0.1/tcp/62620/ipfs/QmWpWmcVJkF6EpmAaVDauku8g1uFGuxPsGP35XZp9GYEqs
/ip4/192.168.2.156/tcp/62620/ipfs/QmWpWmcVJkF6EpmAaVDauku8g1uFGuxPsGP35XZp9GYEqs
node 2 is listening on:
/ip4/127.0.0.1/tcp/10000/ws/ipfs/QmWAQtWdzWXibgfyc7WRHhhv6MdqVKzXvyfSTnN2aAvixX
/ip4/127.0.0.1/tcp/62619/ipfs/QmWAQtWdzWXibgfyc7WRHhhv6MdqVKzXvyfSTnN2aAvixX
/ip4/192.168.2.156/tcp/62619/ipfs/QmWAQtWdzWXibgfyc7WRHhhv6MdqVKzXvyfSTnN2aAvixX
node 3 is listening on:
/ip4/127.0.0.1/tcp/20000/ws/ipfs/QmVq1PWh3VSDYdFqYMtqp4YQyXcrH27N7968tGdM1VQPj1
node 3 failed to dial to node 1 with: No available transport to dial to
node 1 dialed to node 2 successfully
node 2 dialed to node 3 successfully
```

As expected, we created 3 nodes, node 1 with TCP, node 2 with TCP+WebSockets and node 3 with just WebSockets. node 1 -> node 2 and node 2 -> node 3 managed to dial correctly because they shared a common transport, however, node 3 -> node 1 failed because they didn't share any.

## 4. How to create a new libp2p transport

Today there are already 3 transports available, one in the works and plenty to come, you can find these at [interface-transport implementations](https://github.com/libp2p/interface-transport#modules-that-implement-the-interface) list.

Adding more transports is done through the same way as you added TCP and WebSockets. Some transports might offer extra functionalities, but as far as libp2p is concerned, if it follows the interface defined at the [spec](https://github.com/libp2p/interface-transport#api) it will be able to use it.

If you decide to implement a transport yourself, please consider adding to the list so that others can use it as well.

Hope this tutorial was useful. We are always looking to improve it, so contributions are welcome!

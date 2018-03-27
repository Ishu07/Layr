## Batnode Prototype


## To Do

2. Cli integrates with new file retrieval method
4. Shards can be audited by the data owner*
5. Data format of shard transfer is changed to remove the limitations of JSON
6. Kademlia nodes can communicate behind different NATs
7. Kademlia nodes broker connections between BatNodes so that BatNodes can communicate behind different NATs
9. Refactor nested callbacks w/ Async control flow ***
10. Handling node disconnection and restarting ***
11. If a kadnode's batnode isn't listening, don't try to store it (add a check to make sure)

Edge cases for file retrieval and upload:
- batnode offline
- kademlia node offline
- batnode does not have enough storage
- file has been modified
- batnode does not have the requested file

Conventions:

Public seeds always listen on port 80
BatNode CLI servers always listen on port 1800 of localhost
Kademlia Nodes always listen on port 8080
BatNode servers always listen on port 1900

## Specification

#### Upload Single File

kd = kad node
bt = bat node
tkd = target kad node
tbt = target bat node

1. kd performs an iterativeFindNode for <= k kd nodes w/ closest ids to file id
2. kd returns those nodes contacts to bt
3. bt iterates through contacts until the following subroutine succeeds:
4. For each target kad node (tkd):
5. bt asks kd to send a broker_connectionRPC to tkd. Broker connection's payload is bt public conect tuple
6. tkd receives RPC and tells its target bat node (tbt) to send establish_connection tcp request to bt tuple
7. tkd simultaneously sends a success RPC to bt, containing tbt's public tuple
8. bt sends establish_connection tcp request to tbt
9. if bt receives the establish_connection request, it sends a standard store_file message in response
10. if tbt is instead the one to receive establish_connection request, it sends a ready_to_store message in response, and bt responds to that with standard store_file message
11. if bt does not receive a response with X amount of time, move onto the next tkd and try again, else, cease iteration
12. when tbt finishes receiving file, it initiates an iterative store on its tkd

Edge cases:
 - The node doesn't have enough allocated storage
 - The nodes are already connected from previous exchange, so both establish_connection requests go through, causing the file to be sent over twice



#### Retrieve Single File



#### Redefining a file as a group of shards: New versions of Upload and Retrieval




#### Shards and Avoiding Exceeding High Water Marks for JSON-based requests



#### BatNode's public ip/port changes



#### KadNode disconnects and reconnects



#### BatNode-KadNode Pair joins the network


#### BatNodes keep eachother alive



#### Discussion of Chosen NAT Traversal Strategies




### Demos

#### 1: Write a file across two nodes in a LAN.

`node1` will write to `node2`, which resides in another directory.

Starting condition:  
- `demo/batnode1/hosted` has an `example.txt` file.
- Delete any files in `demo/batnode2/hosted`.

1) cd into `demo/batnode1`
2) run `node batnode.js`
3) open a new terminal session and `cd` into `demo/batnode2`
4) run `node batnode.js`

Ending condition: `demo/batnode2/hosted` has a `example.txt-1` file.

#### 2: Distribute shards to multiple server nodes (hardcoded server contacts)

client node distributes shards to two server nodes

Starting condition:
- `demo/demo_upload/client-node/personal` has an `example.txt` and `test.pdf`
- `demo/demo_upload/server-node` and `server-node-2` directories have an `example.txt` and `test.pdf` files

1) `cd` into `demo/demo_upload/server-node`, and run `node batnode.js`. That server is now running.
2) Open a new terminal session, and `cd` into `demo/demo_upload/server-node-2` and do the same.
3) `cd` into `demo/demo_upload/client-node` and run `node batnode.js`

There should now be:
- shards files in `demo/demo_upload/client-node/shards`
- A manifest file in `demo/demo_upload/client-node/manifest`
- 4 (with current settings) shards in `demo/demo_upload/server-node/hosted` and `server-node-2/hosted`.

Run `git clean -xf` to have all the files generated by the demo wiped.


#### 3. Distribute shards to multiple server nodes (batnodes use kadnodes to locate viable hosts)

In this demo, all nodes are comprised of a batNode-kademliaNode pair. A bat node is responsible for transferring and storing file data, while a kademlia node is responsible for locating files and nodes on the network.

To run this demo, you will need three terminal windows.

First, clone the repo, `cd` into it, and install dependencies. Then, follow the commands below:

After npm installing, go into `node_modules/@kadenceproject/lib/node-kademlia.js`

In `node-kademlia.js`, add this to the `listen` method:
`this.use('BATNODE', handlers.batnode.bind(handlers))`
then add these methods to the `KademnliaNode` class itself:

```
  set batNode(node){
    this._batNode = node
  }

  get batNode(){
    return this._batNode
  }


  getOtherBatNodeContact(targetNode, callback) {
    let batcontact = this.batNode.address
    this.send('BATNODE', batcontact, targetNode, callback);
  };
```

After updating `node-kademlia.js`, it is time to update the file located here: `node_modules/@kadenceproject/lib/rules-kademlia.js`

Place the following code into the `KademliaRules` class

```
batnode(req, res) {
  let batnode = this.node.batNode
  res.send(batnode.address)
}
```


In the first terminal window:
1. `cd kad-bat-plugin/node1`
2. `rm -rf db`
3. `rm hosted/*`
4. `node node1.js`

In the second terminal window:
1. `cd kad-bat-plugin/node2`
2. `rm -rf dbb`
3. `rm hosted/*`
4. `node node2.js`

In the third terminal window:
1. `cd kad-bat-plugin/node3`
2. `rm -rf dbbb`
3. `rm manifest/*`
4. `rm shards/*`
5. `node node3.js`

What you will see:

In node3's folder, you should see 8 shards created in the shards folder and a single file generated in the manifest folder. You should then notice that the shards are distributed across nodes 1 and 2. Specifically, the shards in node3 should be found in the `hosted` folders of nodes 1 and 2.

You can then uncomment the `retrieveFile` line in node3.js, but replace the manifest filename with the name of the manifest generated when node3 executed `uploadFile`

#### Demo for CLI sample:
1. Same steps for node1 & node2 in above "3. Distribute shards to multiple server nodes (batnodes use kadnodes to locate viable hosts)".

   In the third terminal window:
    1. `cd kad-bat-plugin/node3`(system can't verify correct file path if you don't go to the client's directory)
    2. `rm -rf dbbb`
    3. `rm manifest/*`
    4. `rm shards/*`
    5. `node clinode3.js`
    
2. Open another(4th) terminal window, select the options to upload/download files while connecting to node1
  - `batchain-sample -u <filePath>`:
    `batchain-sample -u './personal/example.txt'`
  - `batchain-sample -d <manifestFile>`(make sure don't modify the db folders under node1~3) 
3. If your server window keeps running, you can view your current uploaded lists in another window
  - `batchain -l`
4. You can always run `batchain -h` to review available command and options

## Note:

For `npm`: 
1. Run `npm install -g` before running any `batchain` option or command, make sure to 
2. Need to run `npm install -g` when making bin changes
3. If "chalk" is not working for you, run `npm insatll chalk --save` to make the command line more colorful

For `yarn`:
1. Run `yarn link` to create a symbolic link between project directory and executable command
2. Open another terminal window, run `batchain` and you should see:
```
 Usage: batchain [options] [command]


  Commands:

    sample      see the sample nodes running
    help [cmd]  display help for [cmd]

  Options:

    -h, --help  output usage information
    -l, --list  view your list of uploaded files in BatChain network
  ```

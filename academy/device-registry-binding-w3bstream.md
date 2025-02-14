This tutorial demonstrates how to manage device identity and device binding within a DePIN application powered by [W3bstream Devnet](https://w3bstream.com/).

## Introduction

When creating a DePIN application that triggers blockchain logic based on real-world data from devices, it is crucial to address two key aspects:
1. Registering devices that are authorized to send data;
2. Binding authorized assets with their owners;

In traditional IoT applications, device binding enables remote communication between a device and its owner. DePIN applications take this concept a step further, using device binding to identify which wallet address should be rewarded with on-chain assets generated by the device.
In future iterations of W3bstream, these concepts will be integrated in a more streamlined and user-friendly manner. However, while on Devnet, we will demonstrate how these concepts can be implemented in any DePIN application using W3bstream in conjunction with the IoTeX chain.

## Create the project

Let's create a hardhat project, by running the following command: 

```bash
npm install --save-dev @nomiclabs/hardhat-ethers ethers
```

Once the installation is complete, run the following command and follow the instructions: 

```bash
npx hardhat
```

Let's install two other dependencies we'll need, such as Dotenv package and the Open Zeppelin library for smart contracts: 

```bash
npm install dotenv @openzeppelin/contracts 
```

The next thing to do is to navigate to the `scripts` directory, and modify the `deploy.js` file:

```bash
cd scripts && nano deploy.js 
```

by adding the following content: 

```javascript
const { ethers } = require('hardhat');

async function main() {
  const [deployer] = await ethers.getSigners();

  console.log("Deploying contracts with the account:", deployer.address);

  // DevicesRegistry Contract
  const DevicesRegistry = await ethers.getContractFactory("DevicesRegistry");
  console.log("\nDeploying DevicesRegistry contract");
  const devicesRegistry = await DevicesRegistry.deploy();

  console.log("DevicesRegistry Contract")
  console.log("address:", devicesRegistry.address);

  // DeviceBinding Contract
  const DeviceBinding = await ethers.getContractFactory("DeviceBinding");
  console.log("\nDeploying DeviceBinding contract");
  const deviceBinding = await DeviceBinding.deploy(devicesRegistry.address);

  console.log("DeviceBinding Contract")
  console.log("address:", deviceBinding.address);
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
  
```

The next thing to do is to create a `.env` file with the private key you'll use to deploy these contracts: 

```javascript
cd ../ && echo IOTEX_PRIVATE_KEY=<YOUR_PRIVATE_KEY> > .ENV
// e.g. echo IOTEX_PRIVATE_KEY=111111111111111111111111111111111111 > .env
```

Now we'll modify the `hardhat.config.js` file to add the IoTeX Testnet. Run the following command: 

```bash
nano hardhat.config.js
```

And paste the following code:

```javascript
require('dotenv').config();

const IOTEX_PRIVATE_KEY = process.env.IOTEX_PRIVATE_KEY;

module.exports = {
  solidity: "0.8.4",
  networks: {
    hardhat: {
      gas: 8500000,
    },
    testnet: {
      // These are the official IoTeX endpoints to be used by Ethereum clients
      // Testnet https://babel-api.testnet.iotex.io
      // Mainnet https://babel-api.mainnet.iotex.io
      url: `https://babel-api.testnet.iotex.io`,
 
      // Input your Metamask testnet account private key here
      accounts: [`${IOTEX_PRIVATE_KEY}`],
    },
  },
}; 

```

Next, we'll look at the smart contracts, and their roles inside our application.  

## Devices Registry 

We will create a smart contract that enables device manufacturers to register the unique identity of a device on the blockchain. In this scenario, one can envision a situation where each device possesses a private/public key pair. The private key remains hidden within the device, while the public key is registered on the blockchain via the smart contract. This process effectively "whitelists" the device, allowing it to send data to our application.

When a device sends data to the application, it signs the message with its private key. Our application then verifies whether the signature accompanying the message was indeed signed by the private key associated with one of the public keys stored in the `DevicesRegistry` smart contract. The data will only be accepted by the application if the signature matches; otherwise, it will be disregarded.

The `DevicesRegistry` contract is primarily designed to manage the `AuthorizedDevices` mapping, which maps a device's unique public key to a `Device` structure. This mapping is the core component of the contract, as it ensures that only authorized devices can send data to the application. The contract provides functions to register, suspend, activate, and remove devices, all while maintaining the integrity of the `AuthorizedDevices` mapping. This secure system guarantees that only devices with matching signatures can contribute data, thus ensuring the reliability and trustworthiness of our application.

Run the following command: 

```bash
cd contracts && rm Lock.sol && nano DevicesRegistry.sol
```

And paste the code below: 

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

import "@openzeppelin/contracts/access/Ownable.sol";

contract DevicesRegistry is Ownable {
    event DeviceRegistered(bytes32 indexed _deviceId);

    event DeviceDeleted(bytes32 indexed _deviceId);

    event DeviceSuspended(bytes32 indexed _deviceId);

    event DeviceActivated(bytes32 indexed _deviceId);

    struct Device {
        bool isRegistered;
        bool isActive;
    }

    mapping(bytes32 => Device) public AuthorizedDevices;

    constructor() {}

    modifier onlyRegisteredDevice(bytes32 _deviceId) {
        require(
            AuthorizedDevices[_deviceId].isRegistered,
            "Data Source is not registered"
        );
        _;
    }

    modifier onlyUnregisteredDevice(bytes32 _deviceId) {
        require(
            !AuthorizedDevices[_deviceId].isRegistered,
            "Data Source already registered"
        );
        _;
    }

    modifier onlyActiveDevice(bytes32 _deviceId) {
        require(
            AuthorizedDevices[_deviceId].isActive,
            "Data Source is suspended"
        );
        _;
    }

    modifier onlySuspendedDevice(bytes32 _deviceId) {
        require(!AuthorizedDevices[_deviceId].isActive, "Data Source is active");
        _;
    }

    function registerDevice(bytes32 _newDeviceId)
        public
        onlyOwner
        onlyUnregisteredDevice(_newDeviceId)
    {
        AuthorizedDevices[_newDeviceId] = Device(true, true);
        emit DeviceRegistered(_newDeviceId);
    }

    function removeDevice(bytes32 _deviceIdToRemove)
        public
        onlyOwner
        onlyRegisteredDevice(_deviceIdToRemove)
    {
        delete AuthorizedDevices[_deviceIdToRemove];
        emit DeviceDeleted(_deviceIdToRemove);
    }

    function suspendDevice(bytes32 _deviceIdToSuspend)
        public
        onlyOwner
        onlyRegisteredDevice(_deviceIdToSuspend)
        onlyActiveDevice(_deviceIdToSuspend)
    {
        AuthorizedDevices[_deviceIdToSuspend].isActive = false;
        emit DeviceSuspended(_deviceIdToSuspend);
    }

    function activateDevice(bytes32 _deviceIdToActivate)
        public
        onlyOwner
        onlyRegisteredDevice(_deviceIdToActivate)
        onlySuspendedDevice(_deviceIdToActivate)
    {
        AuthorizedDevices[_deviceIdToActivate].isActive = true;
        emit DeviceActivated(_deviceIdToActivate);
    }

    function isAuthorizedDevice(bytes32 _deviceId)
        public
        view
        onlyRegisteredDevice(_deviceId)
        onlyActiveDevice(_deviceId)
        returns (bool)
    {
        return true;
    }
}
```

## Device Binding

The `DeviceBinding` contract is designed to pair an owner with their device, enabling W3bstream to determine the recipient of token rewards generated by a particular device. This functionality is essential for ensuring that the correct wallet address receives the on-chain assets generated by the device.

At the heart of the `DeviceBinding` contract is the `OwnedDevices` mapping, which maps a device's public key to a `Device` structure. The `Device` structure contains the address of the owner to whom the device has been bound. This mapping serves as the foundation for the contract, establishing a clear relationship between devices and their respective owners.

The `DeviceBinding` contract provides various functions for managing device ownership:
1. `bindDevice`: This function allows the contract owner to bind a device to a specific owner's address. It checks if the device is authorized and unbound before establishing the binding.
2. `unbindDevice`: This function allows a device owner or the contract owner to unbind a device from its owner's address. Once unbound, the device can be bound to a new owner if needed.
3. `getDevicesCount`: This function returns the total number of devices registered in the contract.
4. `getDeviceOwner`: This function returns the owner's address associated with a specific device.
5. `getOwnedDevices`: This function returns a list of devices owned by a particular address.

Run the following command: 

```bash
nano DeviceBinding.sol
```

And paste the code below: 

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

import "@openzeppelin/contracts/access/Ownable.sol";
import "./DevicesRegistry.sol";

contract DeviceBinding is Ownable {

    DevicesRegistry public devicesRegistry;

    // Devices ownership management
    struct Device {
        address ownerAddress;
        uint arrayIndex;
    }
    bytes32[] public DeviceIds;
    mapping (bytes32 => Device) public OwnedDevices;

    // Keep track of how many devices an owner owns
    mapping(address => uint) public DevicesCount;

    // Events
    event OwnershipAssigned (bytes32 _deviceId, address _ownerAddress);
    event OwnershipRenounced (bytes32 _deviceId);

    constructor(address _devicesRegistryAddress) {
        devicesRegistry = DevicesRegistry(_devicesRegistryAddress);
    }

    function bindDevice(bytes32 _deviceId, address _ownerAddress) public onlyOwner returns (bool) {
        require(OwnedDevices[_deviceId].ownerAddress == address(0), "device has already been bound");
         require(devicesRegistry.isAuthorizedDevice(_deviceId) == true, "device not authorized");
        
        AddDevice(_deviceId, _ownerAddress);

        emit OwnershipAssigned(_deviceId, _ownerAddress);
        return true;
    }

    function unbindDevice(bytes32 _deviceId) public returns (bool) {
        require(
            (OwnedDevices[_deviceId].ownerAddress == msg.sender) ||
            (msg.sender == this.owner()), 
            "not the device owner");

        removeDevice(_deviceId);

        emit OwnershipRenounced(_deviceId);
        return true;
    }

    function getDevicesCount() public view returns (uint) {
        return DeviceIds.length;
    }

    function getDeviceOwner(bytes32 _deviceId) public view returns (address) {
        return OwnedDevices[_deviceId].ownerAddress;
    }

    function getOwnedDevices(address _ownerAddress) public view returns (bytes32[] memory) {
        bytes32[] memory foundDevices = new bytes32[](DevicesCount[_ownerAddress]);
        uint count = 0;
        Device memory device;

         for (uint i=0; i<DeviceIds.length; i++) {
            device = OwnedDevices[DeviceIds[i]];
            if (device.ownerAddress == _ownerAddress) {
                foundDevices[count] = DeviceIds[i];    
                count++;      
            }      
        }
       
        return foundDevices;
    }

    function AddDevice(bytes32 _deviceId, address _ownerAddress) private {
        OwnedDevices[_deviceId] = Device(_ownerAddress, DeviceIds.length);        
        DeviceIds.push(_deviceId);
        DevicesCount[_ownerAddress]++;
    }

    function removeDevice(bytes32 _deviceId) private {
        Device memory deviceToRemove = OwnedDevices[_deviceId];
        bytes32 lastDeviceID = DeviceIds[DeviceIds.length - 1];
         // Update the last device's arrayIndex, since we will move it in the array
        OwnedDevices[lastDeviceID].arrayIndex = deviceToRemove.arrayIndex;
        // Overwrite the device to delete ID with the last device's ID
        DeviceIds[deviceToRemove.arrayIndex] = DeviceIds[DeviceIds.length - 1];
        // delete the last device's ID from the array
        DeviceIds.pop();
        // decrease the number of devices owned by the owner
        DevicesCount[deviceToRemove.ownerAddress]--;
        // delete the device from the ownerships
        delete OwnedDevices[_deviceId];
    }
}

```

## Deploy the Contracts

To deploy the contracts on IoTeX Testnet simply run: 

```bash
cd ../ && npx hardhat run scripts/deploy.js --network testnet
```

Your log should look something like this: 

![image](https://user-images.githubusercontent.com/77351244/235671464-fbe87e79-41e4-43f0-a71a-eab3378da427.png)

Next we'll see how W3bstream will use these contracts to manage device identity and device binding within our application. 

## Manage Identity and Binding in W3bstream

Now that we have created and deployed the contracts, we need to create an event routing strategy. For more information and for quick tutorials on how to successfully create a strategy, feel free to check out the [W3bstream documentation](https://docs.w3bstream.com/get-started/w3bstream-studio).

We'll be handling our strategy from [W3bstream Devnet Studio](https://dev.w3bstream.com/), which acts as our W3bstream backend control center. For more info, checkout the docs [here](https://docs.w3bstream.com/get-started/running-a-node).

### Smart Contract Monitoring

The first step in our strategy is to create a Smart Contract Minitor. The idea here is for W3bstream to monitor the contracts we just created and trigger a W3bstream event when a new device has been registered (e.g. In the `DevicesRegistry` contract) and when a new device has been bound to its owner (e.g. In the `DeviceBinding` contract). 

The W3bstream Smart Contract Monitor needs a few parameters to work: 
1. **Event Type**: The W3bstream event name refers to the specific event that will be triggered by the monitor when a certain condition on the blockchain is met.
2. **Chain ID**: The Chain ID is a unique identifier for the blockchain where the monitored contract resides. Currently, the monitor supports only the IoTeX Testnet, which has a Chain ID of 4690.
3. **Contract Address**: This is the unique address of the smart contract you want to monitor on the blockchain.
4. **Block Start**: The Block Start is the specific block number where the monitoring process will begin. Typically, this is the block number where the monitored contract was deployed.
5. **Block End**: The Block End is the block number at which the monitoring process will stop. If you set this value to 0, the monitor will continue to run indefinitely.
6. **Topic0**: Topic0 refers to the hashed signature of the event you're monitoring within the target smart contract. To calculate this hash, you can use any online tool of your choice such as [this](https://emn178.github.io/online-tools/keccak_256.html). For more info, follow this quick tutorial on [**"Monitoring Smart Contracts"**](https://docs.w3bstream.com/get-started/w3bstream-studio/triggering-events/monitoring-smart-contracts) from the W3bstream docs. 

Let's go ahead and create a monitor for the DevicesRegistry contract: 

![image](https://user-images.githubusercontent.com/77351244/235672772-5640cd76-5b82-442f-8863-0affacce22c3.png)

Let's add NEW_DEVICE_REGISTERED as Event Type, i.e. The name of the W3bstream event to trigger; Chain ID will stay as 4690, which is IoTeX Testnet, e.g. where we deployed the contract. Let's then add the contract address, which in this example is `0x035708AB26d14735d7016075397E02eA2aa8b2f6`, followed by the block where we want to start the monitor, e.g. `23331289` in this case; we'll put `0` as `Block End` since we want to run this monitor indefinitely. The `topic0`, is the hash of the signature of the event we're monitoring, which in this case is `DeviceRegistered(bytes32 indexed _deviceId);`, so we would use the kekkak256 hashing algorithm on the event signature like this: `DeviceRegistered(bytes32)`.

Now that the contract monitor for the `DevicesRegistry` has been created, we'll go ahead and do the same for the `DeviceBinding` contract. In this case, the contract event we want to monitor is: `event OwnershipAssigned (bytes32 _deviceId, address _ownerAddress);`. We'll call the W3bstream event to trigger: NEW_DEVICE_BOUND. 

Once done, you will see both monitors on your project's page under the **Triggers** tab, like this: 

![image](https://user-images.githubusercontent.com/77351244/235673551-78cb8f2b-a13a-44b0-ae7c-211bbcc57b3c.png)

A couple of quick troubleshooting tips: 
- When inputting the event signature, there should be no spaces
- When adding the `topic0` you should add `0x` to the result you get from the hashing algorithm. 

### Database Configuration

The next step is to configure the W3bstream Database, which will store the payload of the blockchain events we're monitoring. In our case, we'll need two tables: one which will store the ID of the registered device (e.g. `_deviceId`), as well as the two boolean values we have in the `Device` structure to indicate if the device has been registered (e.g. `isRegistered`) and if the device is active (e.g. `isActive`). The other table will store the ID of the device (e.g. `_deviceId`) and the wallet address it has been bound to (e.g. `_ownerAddress`). 

For a quick tutorial on how to configure your W3bstream Database, follow this link [here]. Simply go to the **Data** tab and click on the "+" button to create a new table. The `DevicesRegistry` table should be configured like the image below: 

![image](https://user-images.githubusercontent.com/77351244/235674245-55299df2-174e-4b25-8d46-300fe76ab0c9.png)

You can see that we have simply defined the name of the table, its description and the columns we need. 

Let's go ahead and configure another table for the `DeviceBinding`:

![image](https://user-images.githubusercontent.com/77351244/235674702-ba469a6d-a085-4662-96e3-0bca82b6e8f1.png)

Notes and Troubleshooting tips: 
- Note that we've called the tables just like the contracts they're indexing respectively
- Only user lowercase and underscore when naming your columns

### Create an Applet

The next step in our strategy involves creating an applet to manage data storage in the configured database. The idea here is for W3bstream to index the appropriate contracts and create a new database entry in the appropriate table any time a new device has been registered in our smart contract, or a device has been bound to an owner's address. (These are the events we added to the smart contract monitor earlier). 

W3bstream supports AssemblyScript, Rust and Golang, allowing developers to create W3bstream applets in the language of their choice. To learn more about W3bstream applet KITs and how to use them, check out the documentation [here](https://docs.w3bstream.com/get-started/hello-world/w3bstream-applet-kits). We're going to use AssemblyScript, so go ahead and create a new folder at the root directory of your project called `W3sbtream` and install the W3bstream Applet KIT: 

```typescript
// create a new W3bstream directory
mkdir W3bstream && cd W3bstream

// install AssemblyScript in your project
npm install --save-dev assemblyscript

// initialize the project
npx asinit . -y

// install the W3bstream Applet Kit
npm install @w3bstream/wasm-sdk
```

Whenever you're ready to build, you can run the following command: `npm run asbuild:release`.

From your `W3bstream` directory go into the `Assembly` directory, and update the contents of `index.ts`:

```bash
cd Assembly && nano index.ts
```

With the following code: 

```typescript
import { GetDataByRID, JSON, ExecSQL, Log } from "@w3bstream/wasm-sdk";
import { String, Bool } from "@w3bstream/wasm-sdk/assembly/sql";

export function handle_device_registered(rid: i32): i32 {
Log("New Device Registered Detected: ");
  let message_string = GetDataByRID(rid);
  let message_json = JSON.parse(message_string) as JSON.Obj;
  let topics = message_json.get("topics") as JSON.Arr;
  let device_id = topics._arr[1].toString();
  Log("Device ID: " + device_id);

  // Store the device id in the DB
  Log("Storing device id in DB...");
  let sql = `INSERT INTO "device_registry" (device_id, is_registered, is_active) VALUES (?,?,?);`;
  ExecSQL(sql, [ new String(device_id), new Bool(true), new Bool(true) ]);
  return 0;
}

export function handle_device_binding(rid: i32): i32 {
  Log("New Device Binding Detected: ");
  let message_string = GetDataByRID(rid);
  let message_json = JSON.parse(message_string) as JSON.Obj;
  let topics = message_json.get("topics") as JSON.Arr;
  let device_id = topics._arr[1].toString();
  let owner_address_padded = topics._arr[2] as JSON.Str;

  let owner_address = owner_address_padded.valueOf().slice(26);
  Log("Device ID: " + device_id);
  Log("Owner Address: " + owner_address);

  // Store the device binding in the DB
  Log("Storing device binding in DB...");
  let sql = `INSERT INTO "device_bindings" (device_id, owner_address) VALUES (?,?);`;
  ExecSQL(sql, [ new String(device_id), new String(owner_address)]);
  return 0;
}
```

Now, before getting into the code, let's visualize what the payload will look like. Visualizing it will help inform the logic of the two functions above. (Note: This is the payload of the `OwnershipAssigned` event in the `DeviceBinding` contract). 

```typescript
/* 
"Payload": {
    "address":"0xF40274D96887a3eDe12311cd6603A214c10AC2F5",
    "topics": [
      "0x05d7f0c690676ba31675b45bcdb9ff4c34bb10744ec89d329eacd93c79ecc029",
      "040780ba149d24ee5418084ee193a6be8b3b7cf5329d160fc8902270b342c4fed4"
    ],
    "data":"0x",
    "blockNumber":"0x130cf90",
    "transactionHash":"0x6a3fbe46fca1d958e2a630ae050d07569d1a7e973eb21bddb76b8ddd3f69c901",
    "transactionIndex":"0x0",
    "blockHash":"0xa7479d8e23bfc379527c723a571895d0c8f2f75d0ec6056023b1129e35023751",
    "logIndex":"0x0",
    "removed":false
}
*/
```

Let's now look at the functions in the applet. 

Both functions are, in fact, quite similar: the `GetDataByRID()` method imported from the W3bstream SDK returns a JSON string with the payload shown above. What we need to do is to parse it, and turn it into a JSON object, in order to access the "`topics`" array. 

We know that `topic0` is, in fact, the hash of the signature of the event emitted, so what we need are the following topics, indicating the actual events that were emitted. In the case of the `handle_device_registered` function, we'll only get the `topic1` which is the `device_id` (line 24). After that, we'll store the `device_id` in the database (lines 14 and 15) using the `ExecSQL` from the W3bstream SDK, along with the two boolean values we had determined earlier when we created the table. 

The `handle_device_binding` is very similar, but in this case, we'll also get a `topic2` (the device owner's address, on line 25 ), and we'll add both of these values in the database (lines 33 and 34). 

Time to build the applet with: 

```bash
npm run asbuild:release
```

### Event Routing

The last thing to do to complete the event strategy for our project is to determine the **event routing** logic, e.g. telling W3bstream what to do once a certain W3bstream event is emitted as a consequence of the smart contract events we're monitoring. 

To learn more about events routing, check out the documentation [here](https://docs.w3bstream.com/get-started/w3bstream-studio/creating-strategies). When starting a new project, you'll always have a DEFAULT event, which corresponds to any W3bstream event raised by your app and will subsequently call the **start** handler (assuming your applet includes a **start** function).

Since our project doesn't include a **start** function, we'll go ahead and delete it. We now need to create two event routing strategies, one for the device being bound to an owner's address, and one for a device being registered on-chain. All we need to do is match the W3bstream events raised in the **Smart Contract Monitor**, with the corresponding function in our applet. 

Your event routing strategy should look something like this: 

![image](https://user-images.githubusercontent.com/77351244/235703272-a867c898-a872-4903-8baf-2ccfe66102c0.png)

## Conclusions

Congratulations! You have successfully learned how to manage device identity and device binding in your W3bstream projects. These are very important features in any DePIN application built with W3bstream.  

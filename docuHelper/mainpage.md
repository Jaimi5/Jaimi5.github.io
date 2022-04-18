
\mainpage LoRa Mesher Library
\tableofcontents

## Introduction

The LoRaMesher library implements a distance-vector routing protocol for communicating messages among LoRa nodes. For the interaction with the LoRa radio chip, we leverage RadioLib, a versatile communication library which supports the SX127X LoRa series module available on the hardware we used, among others.

## Dependencies

You can check `library.json` for more details. Basically, we use a modded version of [Radiolib](https://github.com/jgromes/RadioLib) that supports class methods as callbacks and [FreeRTOS](https://freertos.org/index.html) for scheduling maintenance tasks.

## Currently Supported Protocols
- Finding nodes using a simple routing protocol
- Send messages broadcast or to one of the nodes in the routing table
- Send reliable messages. It is accomplish sending ACK and other configuration packets inside LoRa Mesher
- Send large payloads. Since we cannot send packets larger than 222 bytes, we divide the payload and send reliable messages until is all sent/received

## Quick Links
Documentation for most common methods can be found in its reference page. \n
Configuration of the library in \ref Configuration. \n
Some examples of usage of the library in \ref Examples.

## Used technologies
- [RadioLib](https://github.com/jgromes/RadioLib)
- [FreeRTOS](https://freertos.org)
- [Doxygen](https://doxygen.nl)
- [Doxygen Awesome](https://github.com/jothepro/doxygen-awesome-css)

## See also
https://github.com/LoraMesher/LoraMesher


## Copyright  
Copyright (c) 2022 Joan Miquel Solé



\page Examples Examples

In this page you will see a list of examples on how to use the LoRa Mesher library:
- \subpage counter
- \subpage counterAndDisplay

\page counter Counter Example
\tableofcontents

There is, in the source files of this first implementation, an example to test the new functionalities. This example is an implementation of a counter, sending a broadcast message every 10 seconds. To make it easier to understand, we will remove additional functions that are not necessary to make the microcontroller work with the LoRaMesher library.

### Defining the data type and the data counter

As a proof of concept, we will be sending a numeric counter over LoRa. Its value will be incremented every 10 seconds, then te packet will be transmitted. To start, we need to implement the type of data we will use.

In this case, we will only send a `uint32_t`, which is the counter itself.

```
uint32_t dataCounter = 0;
struct dataPacket {
  uint32_t counter = 0;
};

dataPacket* helloPacket = new dataPacket;
```

### LoRaMesh Initialization

To initialize the new implementation, LoRaMesher must be initialized with a function call. This function will be notified every time the microcontroller receives an incoming packet for the user.

```
Serial.begin(115200); //This configuration can be changed
Serial.println("initBoard");

//Get the LoraMesher instance
LoraMesher& radio = LoraMesher::getInstance();

//Initialize the LoraMesher with a processReceivedPackets function
radio.init(processReceivedPackets);
```

We can see that, when starting a new instance of LoRaMesher, we need to pass through a function.

### Received packets function

The function that gets a notification each time the library receives a packet for the user looks like this one:

```
/**
 * @brief Function that process the received packets
 *
 */
void processReceivedPackets(void*) {
    for (;;) {
        /* Wait for the notification of processReceivedPackets and enter blocking */
        ulTaskNotifyTake(pdPASS, portMAX_DELAY);

        //Iterate through all the packets inside the Received User Packets FiFo
        while (radio.getReceivedQueueSize() > 0) {
            Log.trace(F("ReceivedUserData_TaskHandle notify received" CR));
            Log.trace(F("Queue receiveUserData size: %d" CR), radio.getReceivedQueueSize());

            //Get the first element inside the Received User Packets FiFo
            LoraMesher::userPacket<dataPacket>* packet = radio.getNextUserPacket<dataPacket>();

            //Print the data packet
            printDataPacket(packet);

            //Delete the packet when used. It is very important to call this function to release the memory of the packet.
            radio.deletePacket(packet);

        }
    }
}
```

There are some important things we need to be aware of:

1. This function should have a `void*` in the parameters.
2. The function should contain an endless loop.
3. Inside the loop, it is mandatory to have the `ulTaskNotifyTake(pdPASS,portMAX_DELAY)` or equivalent. This function allows the library to notify the function to process pending packets.
4. All the packets are stored inside a private queue.
5. There is a function to get the size of the queue `radio.getReceivedQueueSize()`.
6. You can get the first element with `radio.getNextUserPacket<T>()` where T is the type of your data. 
7. IMPORTANT!!! Every time you call Pop, you need to be sure to call `radio.deletePacket(packet)`. It will free the memory that has been allocated for the packet. If not executed it can cause memory leaks and out of memory errors.

### User data packet

In this section we will show you what there are inside a `userPacket`.
```
struct userPacket {
    uint16_t dst; //Destination address, in this case it should be or local address or BROADCAST_ADDR
    uint16_t src; //Source address
    uint32_t payloadSize = 0; //Payload size in bytes
    T payload[]; //Payload
};
```

Functionalities to use after getting the packet with `LoraMesher::userPacket<T>* userPacket = radio.getNextUserPacket<T>()`:
1. `radio.getPayloadLength(userPacket)` it will get you the payload size in number of T
2. `radio.deletePacket(userPacket)` it will release the memory allocated for this packet.

### Send data packet function

In this section we will present how you can create and send packets. in this example we will use the `dataPacket` data structure.

```
  void loop() {
        helloPacket->counter = dataCounter++;

        //Create packet and send it.
         radio.createPacketAndSend(BROADCAST_ADDR, helloPacket, 1);

        //Wait 10 seconds to send the next packet
        vTaskDelay(10000 / portTICK_PERIOD_MS);
  }
```

In the previous figure we can see that we are using the helloPacket, we add the counter inside it, and we create and send the packet using the LoRaMesher.

The most important part of this piece of code is the function that we call in the `radio.createPacketAndSend()`:

1. The first parameter is the destination, in this case the broadcast address.
2. And finally, the helloPacket (the packet we created) and the number of elements we are sending, in this case only 1 dataPacket.

### Print packet example

When receiving the packet, we need to understand what the Queue will return us. For this reason, in the next subsection, we will explain how to implement a simple packet processing.

```
/**
 * @brief Print the counter of the packet
 *
 * @param data
 */
void printPacket(dataPacket data) {
  Log.verbose(F("Hello Counter received n %X" CR), data.counter);
}

/**
 * @brief Iterate through the payload of the packet and print the counter of the packet
 *
 * @param packet
 */
void printDataPacket(LoraMesher::userPacket<dataPacket>* packet) {
    //Get the payload to iterate through it
    dataPacket* dPacket = packet->payload;
    size_t payloadLength = radio.getPayloadLength(packet);

    for (size_t i = 0; i < payloadLength; i++) {
        //Print the packet
        printPacket(dPacket[i]);
    }
}
```

1. After receiving the packet in the `processReceivedPackets()` function, we call the `printDataPacket()` function.
2. We need to get the payload of the packet using `packet->payload`.
3. We iterate through the `radio.getPayloadLength(packet)`. This will let us know how big the payload is, in dataPackets types, for a given packet. In our case, we always send only one dataPacket.
4. Get the payload and call the `printPacket(dPacket[i])` function, that will print the counter received.

### Source code

\include Counter\src\main.cpp

















\page counterAndDisplay Counter And Display
\tableofcontents

In this example we can see an application that will send a counter to anyone that can listen in one hop.
In addition, there is configured an example on how a display should work using this library. There is one error that the display function of the OLED should be done in the main loop().
In the display we will see the Id of the device, the last counter we sent, the last counter we received and the routing table of the device.

In addition, the counter will be send directly to one of the nodes inside the routing table. Each time it will change the destination.

### Defining the data type and the data counter

As a proof of concept, we will be sending a array which the first element will be the counter over LoRa. Its value will be incremented every 10 seconds, then te packet will be transmitted. To start, we need to implement the type of data we will use.

In this case, we will only send an array of `uint32_t`, which, the first position is the counter itself.

```
uint32_t dataCounter = 0;
struct dataPacket {
    uint32_t counter[35] = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34};
};

dataPacket* helloPacket = new dataPacket;
```

### LoRaMesh Initialization

To initialize the new implementation, LoRaMesher must be initialized with a function call. This function will be notified every time the microcontroller receives an incoming packet for the user.

```
Serial.begin(115200); //This configuration can be changed
Serial.println("initBoard");

//Get the LoraMesher instance
LoraMesher& radio = LoraMesher::getInstance();

//Initialize the LoraMesher with a processReceivedPackets function
radio.init(processReceivedPackets);
```

We can see that, when starting a new instance of LoRaMesher, we need to pass through a function.

### Received packets function

The function that gets a notification each time the library receives a packet for the user looks like this one:

```
/**
 * @brief Function that process the received packets
 *
 */
void processReceivedPackets(void*) {
    for (;;) {
        /* Wait for the notification of processReceivedPackets and enter blocking */
        ulTaskNotifyTake(pdPASS, portMAX_DELAY);

        //Iterate through all the packets inside the Received User Packets FiFo
        while (radio.getReceivedQueueSize() > 0) {
            Log.trace(F("ReceivedUserData_TaskHandle notify received" CR));
            Log.trace(F("Queue receiveUserData size: %d" CR), radio.getReceivedQueueSize());

            //Get the first element inside the Received User Packets FiFo
            LoraMesher::userPacket<dataPacket>* packet = radio.getNextUserPacket<dataPacket>();

            //Print the data packet
            printDataPacket(packet);

            //Delete the packet when used. It is very important to call this function to release the memory of the packet.
            radio.deletePacket(packet);

        }
    }
}
```

There are some important things we need to be aware of:

1. This function should have a `void*` in the parameters.
2. The function should contain an endless loop.
3. Inside the loop, it is mandatory to have the `ulTaskNotifyTake(pdPASS,portMAX_DELAY)` or equivalent. This function allows the library to notify the function to process pending packets.
4. All the packets are stored inside a private queue.
5. There is a function to get the size of the queue `radio.getReceivedQueueSize()`.
6. You can get the first element with `radio.getNextUserPacket<T>()` where T is the type of your data. 
7. IMPORTANT!!! Every time you call Pop, you need to be sure to call `radio.deletePacket(packet)`. It will free the memory that has been allocated for the packet. If not executed it can cause memory leaks and out of memory errors.

### User data packet

In this section we will show you what there are inside a `userPacket`.
```
struct userPacket {
    uint16_t dst; //Destination address, in this case it should be or local address or BROADCAST_ADDR
    uint16_t src; //Source address
    uint32_t payloadSize = 0; //Payload size in bytes
    T payload[]; //Payload
};
```

Functionalities to use after getting the packet with `LoraMesher::userPacket<T>* userPacket = radio.getNextUserPacket<T>()`:
1. `radio.getPayloadLength(userPacket)` it will get you the payload size in number of T
2. `radio.deletePacket(userPacket)` it will release the memory allocated for this packet.

### Send data packet function
In this section we will present how you can create and send packets. in this example we will use the `dataPacket` data structure.

#### Creation of the Task to send

```
/**
 * @brief Setup the Task to create and send periodical messages
 *
 */
void createSendMessages() {
    TaskHandle_t sendLoRaMessage_Handle = NULL;
    BaseType_t res = xTaskCreate(
        sendLoRaMessage,
        "Send LoRa Message routine",
        4098,
        (void*) 1,
        1,
        &sendLoRaMessage_Handle);
    if (res != pdPASS) {
        Log.error(F("Send LoRa Message task creation gave error: %d" CR), res);
        vTaskDelete(sendLoRaMessage_Handle);
    }
}
```

#### Task sendLoraMessage routine

In this function every time is executed it will get the routing table, if there is one or more elements, it will send to the first one the dataCounter with the first element as a counter.

```
/**
 * @brief Every 20 seconds it will send a counter to a position of the dataTable
 *
 */
void sendLoRaMessage(void*) {
    int dataTablePosition = 0;

    for (;;) {
        if (radio.routingTableSize() == 0) {
            vTaskDelay(20000 / portTICK_PERIOD_MS);
            continue;
        }

        if (radio.routingTableSize() <= dataTablePosition)
            dataTablePosition = 0;

        uint16_t addr = radio.routingTable[dataTablePosition].networkNode.address;

        Log.trace(F("Send data packet nº %d to %X (%d)" CR), dataCounter, addr, dataTablePosition);

        dataTablePosition++;

        //Create packet and send it.
        radio.createPacketAndSend(addr, helloPacket, 1);

        //Print second line in the screen
        Screen.changeLineTwo("Send " + String(dataCounter));

        //Increment data counter
        helloPacket->counter[0] = dataCounter++;

        //Print routing Table to Display
        printRoutingTableToDisplay();

        //Wait 20 seconds to send the next packet
        vTaskDelay(20000 / portTICK_PERIOD_MS);
    }
}
```

In the previous figure we can see that we are using the helloPacket, we add the counter inside it, and we create and send the packet using the LoRaMesher.

The most important part of this piece of code is the function that we call in the `radio.createPacketAndSend()`:

1. The first parameter is the destination, in this case the broadcast address.
2. And finally, the helloPacket (the packet we created) and the number of elements we are sending, in this case only 1 dataPacket.

### Print packet example

When receiving the packet, we need to understand what the Queue will return us. For this reason, in the next subsection, we will explain how to implement a simple packet processing.

```
/**
 * @brief Print the counter of the packet
 *
 * @param data
 */
void printPacket(dataPacket* data, uint16_t sourceAddress) {
    char text[32];
    snprintf(text, 32, ("%X-> %d" CR), sourceAddress, data->counter[0]);

    Screen.changeLineThree(String(text));
    Log.verbose(F("Received data nº %d" CR), data->counter[0]);
}

/**
 * @brief Iterate through the payload of the packet and print the counter of the packet
 *
 * @param packet
 */
void printDataPacket(LoraMesher::userPacket<dataPacket>* packet) {
    Log.trace(F("Packet arrived from %X with size %d bytes" CR), packet->src, packet->payloadSize);

    //Get the payload to iterate through it
    dataPacket* dPacket = packet->payload;
    size_t payloadLength = radio.getPayloadLength(packet);

    printPacket(&dPacket[0], packet->src);

    Log.trace(F("---- Payload ---- Payload length in dataP: %d " CR), payloadLength);
    Log.setShowLevel(false);

    for (size_t i = 0; i < payloadLength; i++) {
        Log.verbose(F("Received data nº %d" CR), i);
        Log.verbose(F("%d -- "), i);

        for (size_t j = 0; j < 35; j++) {
            Log.verbose(F("%d, "), dPacket[i].counter[j]);

        }
        Log.verbose(F("" CR));
    }

    Log.setShowLevel(true);
    Log.trace(F("---- Payload Done ---- " CR));

}
```

1. After receiving the packet in the `processReceivedPackets()` function, we call the `printDataPacket()` function.
2. We need to get the payload of the packet using `packet->payload`.
3. We iterate through the `radio.getPayloadLength(packet)`. This will let us know how big the payload is, in dataPackets types, for a given packet. In our case, we always send only one dataPacket.
4. Get the payload and call the `printPacket(dPacket[i])` function, that will print the counter received.

### Source code

\include CounterAndDisplay\src\main.cpp
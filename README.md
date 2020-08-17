# msgpackr

msgpackr is an extremely fast MessagePack NodeJS/JavaScript implementation. At the time of writing, it is several times faster than any other known implementations, faster than Avro (for JS), and generally even faster than native JSON.stringify/parse. It also includes an optional record extension, for defining record structures that makes MessagePack even faster and more compact, often over twice as fast as even native JSON methods and many times faster any other JS implementations.

## Basic Usage

Install with:

```
npm install msgpackr
```
And import or require it for basic parse and serialize functions:
```
import { parse, serialize } from 'msgpackr';
let serializedAsBuffer = serialize(value);
let data = parse(serializedAsBuffer);
```
This `serialize` function will generate standard MessagePack without any extensions that should be compatible with any standard MessagePack parser. It will serialize JavaScript objects as MessagePack maps by default. The `parse` function will deserialize MessagePack maps as an `Object` with the properties from the map.

# Record / Object Structures
There is a critical different between maps or dictionaries that hold an arbitrary set of keys and values, and record or object structures that have a well-defined set of fields that may have many instances using that structure. By using the record extension, this distinction is preserved in MessagePack and the encoding can reuse structures not only provides better type preservation, but can yield signficantly more compact encodings and increases parsing/deserialization performance by 2-3x. msgpackr automatically generates record definitions that are reused and referenced by objects with the same structure. There are a number of ways to use this to our advantage. For large object structure with a repeating objects with similar structures, the record can yield benefits. To use the record structures extension, we create a new Serializer or SerializerStream instance. By default a new Serializer or SerializerStream instance will have the record extension enabled:
```
import { Serializer } from 'msgpackr';
let serializer = Serializer();
serializer.serialize(myBigData);

```

One way to further leverage the benefits of the msgpackr record structures is to use streams that naturally allow for data to reuse previous record structures. We can use the SerializerStream instance that will have the record extension enabled:

```
import { SerializerStream } from 'msgpackr';
let stream = SerializerStream();
stream.write(myData);

```
Or for a full example of sending and receiving data on a stream
```
import { SerializerStream } from 'msgpackr';
let stream1 = SerializerStream();
let stream2 = SerializerStream();
stream1.pipe(stream2); // we just piping to our own stream, but normally you would send and receive over some type of inter-process or network connection.
stream1.write(myData);
stream2.on('data', (data) => {
	// received data
}

```

## Shared Record Structures
Another valuable way of using msgpackr is for storing data, such as in a database or in files. If a number of objects are being stored with common data structures, a shared structure can be used to greatly improve data storage and deserialization efficiency. We just need to provide a to store the generated shared structure so it is available to deserialize stored data in the future.

```
import { Serializer } from 'msgpackr';
let serializer = Serializer({
	getStructures() {
		return parse(readFileSync('my-shared-structures.mp')) || [];
	},
	saveStructures() {
		writeFileSync('my-shared-structures.mp')
	}
	structures: []
});
serializer.serialize(myBigData);

```



### resetMemory
During the serialization process, data is written to buffers. Allocating new buffers is a relatively expensive process, and the `resetMemory` method can help allow reuse buffers that will further improve performance. The `resetMemory` method can be called when previously created buffer(s) are no longer needed. For example, if we serialized an object, and wrote it to a database, we could indicate that we are done:
```
let buffer = serializer.serialize(data);
writeToStorageSync(buffer);
// finished with buffer, we can reset the memory on our serializer now:
serializer.resetMemory()
// future serialization can now reuse memory for better performance
```





operation                                                  |   op   |   ms  |  op/s
---------------------------------------------------------- | ------: | ----: | -----:
buf = Buffer(JSON.stringify(obj));                         | 1215400 |  5000 | 243080
obj = JSON.parse(buf);                                     | 1290100 |  5000 | 258019
msgpackr {objectsAsMaps: true}: serializer.serialize(obj); | 2936100 |  5000 | 587220
msgpackr {objectsAsMaps: true}: serializer.parse(buf);     | 1729800 |  5000 | 345960
msgpackr w/ shared structures: serializer.serialize(obj);  | 3393200 |  5000 | 678640
msgpackr w/ shared structures: serializer.parse(buf);      | 4491900 |  5000 | 898380
buf = require("msgpack-lite").encode(obj);                 |  430300 |  5000 |  86060
obj = require("msgpack-lite").decode(buf);                 |  268300 |  5001 |  53649
buf = require("notepack").encode(obj);                     | 1113200 |  5000 | 222640
obj = require("notepack").decode(buf);                     |  543800 |  5000 | 108760
require("what-the-pack")... encoder.encode(obj);           | 1019800 |  5000 | 203960
require("what-the-pack")... encoder.decode(buf);           |  535200 |  5000 | 107040
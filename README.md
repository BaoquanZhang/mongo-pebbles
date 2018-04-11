## PebblesDB Storage Engine Module for MongoDB

### PebblesDB
PebblesDB is a write-optimized key-value store which is built using the novel FLSM (Fragmented Log-Structured Merge Tree) data structure. FLSM is a modification of standard LSM data structure which aims at achieving higher write throughput and lower write amplification without compromising on read throughput.
PebblesDB is built by modifying [HyperLevelDB](https://github.com/rescrv/HyperLevelDB) which, in turn, is built on top of [LevelDB](https://github.com/google/leveldb). PebblesDB is API compatible with HyperLevelDB and LevelDB. Thus, PebblesDB is a drop-in replacement for LevelDB and HyperLevelDB. The full paper on PebblesDB can be found [here](http://www.cs.utexas.edu/~vijay/papers/sosp17-pebblesdb.pdf "PebblesDB SOSP'17").

Please [cite](http://www.cs.utexas.edu/~vijay/bibtex/sosp17-pebblesdb.bib) the following paper if you use PebblesDB: [PebblesDB: Building Key-Value Stores using Fragmented Log-Structured Merge Trees](http://www.cs.utexas.edu/~vijay/papers/sosp17-pebblesdb.pdf). Pandian Raju, Rohan Kadekodi, Vijay Chidambaram, Ittai Abraham. [SOSP 17](https://www.sigops.org/sosp/sosp17/). [Bibtex](http://www.cs.utexas.edu/~vijay/bibtex/sosp17-pebblesdb.bib)

The github link to PebblesDB is [here](https://github.com/utsaslab/pebblesdb.git). 

### mongo-pebbles
mongo-pebbles is a wrapper built to allow MongoDB use PebblesDB as its backend storage engine. Currently, MongoDB uses the WiredTiger key-value store as its default storage engine. 

mongo-pebbles is a modification of [mongo-rocks](https://github.com/mongodb-partners/mongo-rocks.git), thus making PebblesDB compatible with MongoDB

### Dependencies
PebblesDB requires libsnappy and libtool. To install on Linux, please use sudo apt-get install libsnappy-dev libtool.

mongo-pebbles has been built, compiled and tested with g++-5. It may not work with other versions of g++ and other C++ compilers.

As of now, mongo-pebbles is compatible only with MongoDB v3.4.4, and is not guaranteed to work for a different version of MongoDB. 

### Installation Steps

Execute this series of commands to compile MongoDB v3.4.4 with PebblesDB storage engine:
    
    # get pebblesdb
    git clone https://github.com/utsaslab/pebblesdb.git
    # compile pebblesdb
    cd pebblesdb; autoreconf -i; ./configure; make; make install; ldconfig
    # get mongo
    git clone https://github.com/mongodb/mongo.git
    # mongo use version 3.4.4
    cd mongo; git checkout r3.4.4
    # get mongo-pebbles
    git clone https://github.com/mongodb-partners/mongo-rocks
    # add pebblesdb module to mongo
    mkdir -p mongo/src/mongo/db/modules/
    ln -sf ~/mongo-rocks mongo/src/mongo/db/modules/pebbles
    # compile mongo
    cd mongo; scons
    # set ulimit for supporting maximum number of open files
    ulimit -n 5000

Start `mongod` using the `--storageEngine=pebblesdb` option.

**NOTE:* Please edit `dynamic_syslibdeps.append("/usr/local/lib/libpebblesdb.so")` in `SConscript` to point 
to your `libpebblesdb.so` path.

### More information

To use this module, it has to be linked from `mongo/src/mongo/db/modules`. The build system will automatically recognize it. In the `mongo` repository directory do the following:

    mkdir -p src/mongo/db/modules/
    ln -sf ~/mongo-rocks src/mongo/db/modules/pebbles

### Note
For reproducing the results mentioned in the [SOSP Paper](http://www.cs.utexas.edu/~vijay/papers/sosp17-pebblesdb.pdf), the following parameters have to be set:

1. Memtable size = 16MB
2. Compression disabled
3. Table cache size = 3,000

These parameters have been set by default through mongo-pebbles.

### Changes from mongo-rocks
1. Column Families provide a way to logically partition the database in RocksDB, where users can create and remove Column Families on the fly. As of now, PebblesDB does not provide support for Column Families. Hence, mongo-rocks had to be modified to remove support of Column Families for supporting PebblesDB. 

2. PebblesDB and RocksDB have many parameters in common like maximum background compactions, level 0 slowdown trigger, rate limiter, etc. However, these parameters cannot be specified as a command line parameter while booting up PebblesDB, as opposed to RocksDB where users can specify these parameters. Hence, these configuration options cannot be set from mongo-pebbles, and mongo-rocks had to be modified to remove setting of these parameters when starting the key-value store.

3. A WriteBatch is a set of writes to the key-value store that have not yet been committed, but are guaranteed to be atomic. Both RocksDB and PebblesDB support the concept of a WriteBatch. However, PebblesDB, as opposed to RocksDB, does not have a WriteBatch Iterator, which is used to iterate through the records in a WriteBatch. The implementation of a WriteBatch Iterator for PebblesDB has thus been added to mongo-pebbles, which was absent in mongo-rocks. A WriteBatch Iterator is implemented with the help of a standard C++ unordered map. Each entry into the WriteBatch Iterator contains the data to be written, along with the type of the write (eg: data write or a metadata write). We understand that this is not the most optimal implementation for a WriteBatch Iterator, feel free to suggest improvements in the implementation of this Iterator. The code for the WriteBatch iterator is present in the file src/writebatch_iterator.h

### Reach out

If you have any issues with mongo-pebbles, feel free to drop an e-mail on the following address: `vijay@cs.utexas.edu`

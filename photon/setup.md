## 1. Download Asia Data Dump

First, download the raw data from GraphHopper. This file is approximately **2.3GB**.

```
wget https://download1.graphhopper.com/public/asia/photon-dump-asia-1.0-latest.jsonl.zst
```

## 2. Import and Filter Data
Run the import command. We use zstd to decompress the data on the fly and pipe it into Photon, filtering specifically for Malaysia (-country-codes my).

It might takes roughly **30 minutes** for normal version, faster for the thread-optimized command.

### Normal Version:
```
zstd --stdout -d photon-dump-asia-1.0-latest.jsonl.zst | java -jar photon-1.0.0.jar import -import-file - -country-codes my
```

#### /OR

### Thread-Optimized Command (change the number of active processor)

```
nice -n 19 zstd -d -c -T10 photon-dump-asia-1.0-latest.jsonl.zst | \
java -Xmx10G -XX:ActiveProcessorCount=8 -jar photon-1.0.0.jar \
import -import-file - -country-codes my
```

Note: This creates a directory named photon_data which contains the search index.

##  3. Start the Server

```
sudo docker compose up
```

## 4. Test the container
```
curl "http://localhost:2322/api?q=kuala+lumpur"
```

# Astro-DB

## Preliminary Information

We're constructing a prototype for a Mongo database that can handle 100 million event data insertions a night while maintaining fast cone search speeds. To achieve this, we're testing database sharding strategies and the utility of the 2dsphere index.
To shard a database in MongoDB, one must specify a shard key, which must correspond to an index on that database. The 2dsphere index and the sharding index are different indexes with different goals. The 2dsphere index maps event locations on a sphere, which is useful for cone searches.

## Cone Searches

All sharded databases tested within this section and the "Writes" section are sharded using an id hash index. There is more data on shard index selection below.
The index referred to in the images is a 2dsphere index built from the array [ra - 180, dec] given by each event.
The Mongo cone search query is as follows: {
	sphere_coordinates: {
		$geoWithin: {
			$centerSphere: [[(ra - 180), (dec)], radius in Rad]
		}
	}
}

<img width="800" height="500" alt="ConeSearch_0 1_Rad" src="https://github.com/user-attachments/assets/fa2feed5-b862-47ae-bc2e-2f6ea01cf189" />

We see that, for decently small cone searches, the 2dsphere index itself reduces query time by almost two orders of magnitude. Sharding helps as well, decreasing query time substantially both with and without an index. 

<img width="800" height="500" alt="ConeSearch_1_Rad" src="https://github.com/user-attachments/assets/6c144617-7f96-458c-bb38-5a023d68b8ed" />

As the radius (in radians of the sphere we're searching on) of the cone search increases, the 2dsphere index doesn't reduce the query time as much. This makes sense, as the difference between an intelligent search that targets a large portion of the data and a naive search across all the data is small.

## Writes

Because all shards reside on a single laptop SSD, insertion data in this section does not reflect real-world, multi-machine sharded performance.

<img width="800" height="500" alt="Sequential_writes_160K" src="https://github.com/user-attachments/assets/8d7bef9b-6e48-4f26-8980-1a01db97e647" />
<img width="800" height="500" alt="Bulk_writes_160K_2" src="https://github.com/user-attachments/assets/495106d5-019e-4584-bade-38f43aaf8feb" />


## HEALPix Tile Sharding Versus Id Hash Sharding

Given sky coordinates, calculating a HEALPix tiling is very fast using Astropy. Thus, HEALPix tile sharding has only slightly more overhead cost than id hash sharding. However, HEALPix sharding introduces cone search inefficiencies displayed below. 

The query: {
	sphere_coordinates: {
		$geoWithin: {
			$centerSphere: [[50, -50], 0.1]
		}
	}
}

### id hash sharding
<img width="2400" height="576" alt="id_hash_sharding cone search 0 1 rad" src="https://github.com/user-attachments/assets/4809f9fd-0bc7-46a0-8166-eacd57369512" />

### HEALPix tile sharding
<img width="2431" height="580" alt="HEALPix_sharding cone search 0 1 rad" src="https://github.com/user-attachments/assets/3e8f51a7-137a-472c-8994-d0c7f043e6ec" />

Note: there is a slight discrepancy in the number of documents returned, with the HEALPix sharded database returning slightly more. This is because I used this particular database when performing a small insertion test. This will have only impacted the final result seen here by a couple ms at most. 

### Comparing query runtime
<img width="800" height="500" alt="id hash sharding vs HEALPix tile sharding cone search 0 1 rad" src="https://github.com/user-attachments/assets/f5932635-831c-49d4-90d4-aaba4b2cda4c" />

The id hash sharding performs cone searches significantly quicker than worst-case HEALPix tile sharding. This is likely because id hashing stores events on random shards without considering location, whereas HEALPix tile sharding is likely to place neighboring events in the same tile, on the same shard. Cone searches on one shard are slower than cone searches spread conmcurrently across multiple shards.

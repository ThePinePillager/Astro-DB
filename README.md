# Astro-DB

## Preliminary Information

We're constructing a prototype for a Mongo database that can handle 100 million event data insertions a night while maintaining fast cone search speeds. To achieve this, 

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

Data in this section isn't likely indicative of sharded database performance when shards are distributed across multiple machines. All shards for each database live on a single SSD connected to a single laptop, which partially defeats the point of sharding a database.

<img width="800" height="500" alt="Sequential_writes_160K" src="https://github.com/user-attachments/assets/8d7bef9b-6e48-4f26-8980-1a01db97e647" />
<img width="800" height="500" alt="Bulk_writes_160K_2" src="https://github.com/user-attachments/assets/495106d5-019e-4584-bade-38f43aaf8feb" />





## HEALPix Tile Sharding Versus Id Hash Sharding




To test the performance difference between the same database sharded on HEALPix tiles 

<img width="2400" height="576" alt="id_hash_sharding cone search 0 1 rad" src="https://github.com/user-attachments/assets/4809f9fd-0bc7-46a0-8166-eacd57369512" />
<img width="2431" height="580" alt="HEALPix_sharding cone search 0 1 rad" src="https://github.com/user-attachments/assets/3e8f51a7-137a-472c-8994-d0c7f043e6ec" />


<img width="800" height="500" alt="id hash sharding vs HEALPix tile sharding cone search 0 1 rad" src="https://github.com/user-attachments/assets/f5932635-831c-49d4-90d4-aaba4b2cda4c" />

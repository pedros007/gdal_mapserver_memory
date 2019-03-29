# GDAL/MapServer Memory Utilization

I run MapServer in a Docker container. I noticed its memory
utilization climbed much higher than expected. This only seems to
happen when accessing files using `/vsicurl/` and `/vsis3/`
[network-based file systems](https://www.gdal.org/gdal_virtual_file_systems.html)
This only started happening after I built MapServer with GDAL > 2.2.4
(tested with 2.3.0, 2.3.2 & 2.4.1 and MapServer-7.0.7 and
MapServer-7.2.2). This repo is an attempt to deomonstrate the problem.

1. Build MapServer with gdal-2.2.4: ```docker build -t gdal_mapserver_memory:2.2.4 -f Dockerfile2.2.4 .```
1. Start MapServer: ```docker run --rm -it -p 8224:80 --name gdal2.2.4 gdal_mapserver_memory:2.2.4```
2. In a separate shell, watch the memory utilization of docker: ```docker stats gdal2.2.4```
3. Note the memory usage of about 24 MB
4. Request a map tile: http://localhost:8224/mapserv?LAYERS=my_layer&FORMAT=image%2Fpng&MAP=/app/mapfiles/indonesian_redcross.map&TRANSPARENT=true&SERVICE=WMS&VERSION=1.1.1&REQUEST=GetMap&STYLES=&SRS=EPSG%3A3857&BBOX=13894417.251632,-1013249.2468073,13895028.747858,-1012637.7505811&WIDTH=512&HEIGHT=512
5. Note memory usage of about 49 MB
6. After serving many different WMS requests, memory usage stays around 500 MB.

1. Build MapServer with gdal-2.3.0: ```docker build -t gdal_mapserver_memory:2.3.0 -f Dockerfile2.3.0 .```
1. Start MapServer: ```docker run --rm -it -p 8230:80 --name gdal2.2.4 gdal_mapserver_memory:2.2.4```
2. In a separate shell, watch the memory utilization of docker: ```docker stats gdal2.2.4```
3. Note the memory usage of about 23 MB
4. Request a map tile: http://localhost:8230/mapserv?LAYERS=my_layer&FORMAT=image%2Fpng&MAP=/app/mapfiles/indonesian_redcross.map&TRANSPARENT=true&SERVICE=WMS&VERSION=1.1.1&REQUEST=GetMap&STYLES=&SRS=EPSG%3A3857&BBOX=13894417.251632,-1013249.2468073,13895028.747858,-1012637.7505811&WIDTH=512&HEIGHT=512
5. Note memory usage of about 214 MB
6. After serving many different WMS requests, memory usage grows to a several GB.


# Valgrind & Resident Set Size
I tried running valgrind on `mapserv` and noticed the total number of
bytes allocated is about 4x greater using MapServer with GDAL > 2.2.4.

## gdal-2.2.4

```bash
docker run --rm gdal_mapserver_memory:2.2.4 sh -c 'valgrind /usr/local/bin/mapserv QUERY_STRING="LAYERS=my_layer&FORMAT=image%2Fpng&MAP=/app/mapfiles/indonesian_redcross.map&TRANSPARENT=true&SERVICE=WMS&VERSION=1.1.1&REQUEST=GetMap&STYLES=&SRS=EPSG%3A3857&BBOX=13894417.251632,-1013249.2468073,13895028.747858,-1012637.7505811&WIDTH=512&HEIGHT=512" > /dev/null'
==8== Memcheck, a memory error detector
==8== Copyright (C) 2002-2015, and GNU GPL'd, by Julian Seward et al.
==8== Using Valgrind-3.12.0.SVN and LibVEX; rerun with -h for copyright info
==8== Command: /usr/local/bin/mapserv QUERY_STRING=LAYERS=my_layer&FORMAT=image%2Fpng&MAP=/app/mapfiles/indonesian_redcross.map&TRANSPARENT=true&SERVICE=WMS&VERSION=1.1.1&REQUEST=GetMap&STYLES=&SRS=EPSG%3A3857&BBOX=13894417.251632,-1013249.2468073,13895028.747858,-1012637.7505811&WIDTH=512&HEIGHT=512
==8==
==8==
==8== HEAP SUMMARY:
==8==     in use at exit: 115,399 bytes in 726 blocks
==8==   total heap usage: 353,577 allocs, 352,851 frees, 164,011,998 bytes allocated
==8==
==8== LEAK SUMMARY:
==8==    definitely lost: 0 bytes in 0 blocks
==8==    indirectly lost: 0 bytes in 0 blocks
==8==      possibly lost: 0 bytes in 0 blocks
==8==    still reachable: 115,399 bytes in 726 blocks
==8==         suppressed: 0 bytes in 0 blocks
==8== Rerun with --leak-check=full to see details of leaked memory
==8==
==8== For counts of detected and suppressed errors, rerun with: -v
==8== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
```

RSS
```bash
docker run --rm gdal_mapserver_memory:2.2.4 sh -c 'time -v /usr/local/bin/mapserv QUERY_STRING="LAYERS=my_layer&FORMAT=image%2Fpng&MAP=/app/mapfiles/indonesian_redcross.map&TRANSPARENT=true&SERVICE=WMS&VERSION=1.1.1&REQUEST=GetMap&STYLES=&SRS=EPSG%3A3857&BBOX=13894417.251632,-1013249.2468073,13895028.747858,-1012637.7505811&WIDTH=512&HEIGHT=512" > /dev/null'
        ...
        Maximum resident set size (kbytes): 57656
		...
```

## gdal-2.3.0

```bash
docker run --rm gdal_mapserver_memory:2.3.0 sh -c 'valgrind /usr/local/bin/mapserv QUERY_STRING="LAYERS=my_layer&FORMAT=image%2Fpng&MAP=/app/mapfiles/indonesian_redcross.map&TR
> ANSPARENT=true&SERVICE=WMS&VERSION=1.1.1&REQUEST=GetMap&STYLES=&SRS=EPSG%3A3857&BBOX=13894417.251632,-1013249.2468073,13895028.747858,-1012637.7505811&WIDTH=512&HEIGHT=512" > /dev/null'
==8== Memcheck, a memory error detector
==8== Copyright (C) 2002-2015, and GNU GPL'd, by Julian Seward et al.
==8== Using Valgrind-3.12.0.SVN and LibVEX; rerun with -h for copyright info
==8== Command: /usr/local/bin/mapserv QUERY_STRING=LAYERS=my_layer&FORMAT=image%2Fpng&MAP=/app/mapfiles/indonesian_redcross.map&TR
==8== ANSPARENT=true&SERVICE=WMS&VERSION=1.1.1&REQUEST=GetMap&STYLES=&SRS=EPSG%3A3857&BBOX=13894417.251632,-1013249.2468073,13895028.747858,-1012637.7505811&WIDTH=512&HEIGHT=512
==8==
==8==
==8== HEAP SUMMARY:
==8==     in use at exit: 115,399 bytes in 726 blocks
==8==   total heap usage: 1,715,942 allocs, 1,715,216 frees, 667,932,430 bytes allocated
==8==
==8== LEAK SUMMARY:
==8==    definitely lost: 0 bytes in 0 blocks
==8==    indirectly lost: 0 bytes in 0 blocks
==8==      possibly lost: 0 bytes in 0 blocks
==8==    still reachable: 115,399 bytes in 726 blocks
==8==         suppressed: 0 bytes in 0 blocks
==8== Rerun with --leak-check=full to see details of leaked memory
==8==
==8== For counts of detected and suppressed errors, rerun with: -v
==8== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
```

RSS
```bash
docker run --rm gdal_mapserver_memory:2.3.0 sh -c 'time -v /usr/local/bin/mapserv QUERY_STRING="LAYERS=my_layer&FORMAT=image%2Fpng&MAP=/app/mapfiles/indonesian_redcross.map&TRANSPARENT=true&SERVICE=WMS&VERSION=1.1.1&REQUEST=GetMap&STYLES=&SRS=EPSG%3A3857&BBOX=13894417.251632,-1013249.2468073,13895028.747858,-1012637.7505811&WIDTH=512&HEIGHT=512" > /dev/null'
        ...
        Maximum resident set size (kbytes): 228232
		...
```

# Data Source

I fetched a TIF from this QGIS-tutorial https://www.cogeo.org/qgis-tutorial.html  `validate_cloud_optimized_geotiff.py` said it wasn't a COG, so I made one at https://s3.amazonaws.com/pschmitt-test-public/b378087a-c2a5-43a0-abec-71fcfb051150_cog.tif
The Mapfile is available at mapfiles/indonesian_redcross.map

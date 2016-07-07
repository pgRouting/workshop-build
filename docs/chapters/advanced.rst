..
   ****************************************************************************
    pgRouting Workshop Manual
    Copyright(c) pgRouting Contributors

    This documentation is licensed under a Creative Commons Attribution-Share
    Alike 3.0 License: http://creativecommons.org/licenses/by-sa/3.0/
   ****************************************************************************

.. _advanced:

Advanced Routing Queries
===============================================================================

* :ref:`intro`

  * :ref:`p-7` Single Driver Routing.

* :ref:`modify` 

  * :ref:`p-8` Single Driver Routing encourage on fast road.
  * :ref:`p-9` Restricted Access.

.. _intro:

Introduction
...............

A query for routing vehicles differs from routing pedestrians explained in the :ref:`chapter about routing algorithms <routing>`.

* There is a `reverse_cost` involved, and the graph is directed.
* This is due to the fact that there are roads that are one way, and depending on the geometry, the valid way is the

  * (source, target) segment (`cost` >= 0 and `reverse_cost` < 0)
  * (target, source) segment (`cost` < 0 and `reverse_cost` >= 0)
  * So a `wrong way` is indicated with a negative value and is not inserted in the graph for processing.

For two way roads `cost` >= 0 and `reverse_cost` >= 0 and their values can be different.
For example its faster going down hill on a sloped road.

In general cost doesn't need to be length, cost can be almost anything, for example time, slope, surface, road type, etc..
Or it can be a combination of multiple parameters.

Of course pgRouting allows you all kind of SQL that is possible with PostgreSQL/PostGIS.

.. note::

    The following dump file contains the data generated with the latest version of osm2pgrouting

.. code-block:: bash

    # Optional: Drop database
    dropdb -U user pgrouting-workshop

    # Load database dump file
    psql -U user -d postgres -f ~/Desktop/pgrouting-workshop/data/sampledata_routing.sql


Using the psql client, verify the database tables:

.. rubric:: Run: ``\dx ways``
                                                            Table "public.ways"
.. code-block:: sql

          Column       |           Type            |                     Modifiers                      | Storage  | Stats target | Description 
    -------------------+---------------------------+----------------------------------------------------+----------+--------------+-------------
     gid               | bigint                    | not null default nextval('ways_gid_seq'::regclass) | plain    |              | 
     class_id          | integer                   | not null                                           | plain    |              | 
     length            | double precision          |                                                    | plain    |              | 
     length_m          | double precision          |                                                    | plain    |              | 
     name              | text                      |                                                    | extended |              | 
     source            | bigint                    |                                                    | plain    |              | 
     target            | bigint                    |                                                    | plain    |              | 
     x1                | double precision          |                                                    | plain    |              | 
     y1                | double precision          |                                                    | plain    |              | 
     x2                | double precision          |                                                    | plain    |              | 
     y2                | double precision          |                                                    | plain    |              | 
     cost              | double precision          |                                                    | plain    |              | 
     reverse_cost      | double precision          |                                                    | plain    |              | 
     cost_s            | double precision          |                                                    | plain    |              | 
     reverse_cost_s    | double precision          |                                                    | plain    |              | 
     rule              | text                      |                                                    | extended |              | 
     one_way           | integer                   |                                                    | plain    |              | 
     maxspeed_forward  | integer                   |                                                    | plain    |              | 
     maxspeed_backward | integer                   |                                                    | plain    |              | 
     osm_id            | bigint                    |                                                    | plain    |              | 
     source_osm        | bigint                    |                                                    | plain    |              | 
     target_osm        | bigint                    |                                                    | plain    |              | 
     priority          | double precision          | default 1                                          | plain    |              | 
     the_geom          | geometry(LineString,4326) |                                                    | main     |              | 
    Indexes:
        "ways_pkey" PRIMARY KEY, btree (gid)
        "ways_gdx" gist (the_geom)
        "ways_source_idx" btree (source)
        "ways_source_osm_idx" btree (source_osm)
        "ways_target_idx" btree (target)
        "ways_target_osm_idx" btree (target_osm)

.. rubric:: Run: ``\dx ways_vertices_pgr``

.. code-block:: sql

                                                    Table "public.ways_vertices_pgr"
      Column  |         Type         |                           Modifiers                            | Storage | Stats target | Description 
    ----------+----------------------+----------------------------------------------------------------+---------+--------------+-------------
     id       | bigint               | not null default nextval('ways_vertices_pgr_id_seq'::regclass) | plain   |              | 
     osm_id   | bigint               |                                                                | plain   |              | 
     cnt      | integer              |                                                                | plain   |              | 
     chk      | integer              |                                                                | plain   |              | 
     ein      | integer              |                                                                | plain   |              | 
     eout     | integer              |                                                                | plain   |              | 
     lon      | numeric(11,8)        |                                                                | main    |              | 
     lat      | numeric(11,8)        |                                                                | main    |              | 
     the_geom | geometry(Point,4326) |                                                                | main    |              | 
    Indexes:
        "ways_vertices_pgr_pkey" PRIMARY KEY, btree (id)
        "vertex_id" UNIQUE CONSTRAINT, btree (osm_id)
        "ways_vertices_pgr_gdx" gist (the_geom)
        "ways_vertices_pgr_osm_id_idx" btree (osm_id)
    
.. _p-7:

Exercise 7
...........................

.. rubric:: Single Driver Routing

* Driver “I am in vertex 30 and want to Drive to vertex 60.”

.. rubric:: Problem description

* The driver wants to go from vertex 30 to vertex 60.
* The driver’s cost is in terms of length. In this case length is in degrees.
* osm2pgrouting cost and reverse_cost columns have lenght in degrees, but a negative length is used to indicate `wrong way`

.. rubric:: Query

.. code-block:: sql

    SELECT * FROM pgr_dijkstra('
        SELECT gid AS id,
            source,
            target,
            cost,
            reverse_cost
            FROM ways',
         30, 60);


.. rubric:: Query Result

.. code-block:: sql

     seq | path_seq | node  | edge  |         cost         |       agg_cost       
    -----+----------+-------+-------+----------------------+----------------------
       1 |        1 |    30 | 59650 | 0.000196604399745105 |                    0
       2 |        2 | 22440 | 64869 |  0.00257000873345393 | 0.000196604399745105
       3 |        3 | 15707 | 70578 |  0.00222916106640424 |  0.00276661313319903
    ...
      56 |       56 | 43766 | 25394 |  0.00113171983281568 |   0.0524679252328294
      57 |       57 |    60 |    -1 |                    0 |   0.0535996450656451
    (57 rows)



.. _modify:

Modifying Costs
-------------------------------------------------------------------------------

In "real" networks there are different limitations or preferences for different road types for example. In other words, we don't want to get the *shortest* but the **cheapest** path - a path with a minimal cost. There is no limitation in what we take as costs.

When we convert data from OSM format using the osm2pgrouting tool, we get two additional tables for road ``osm_way_types`` and road ``osm_way_classes``:

.. note::

    We switch now to the database we previously generated with osm2pgrouting. From within PostgreSQL shell this is possible with the ``\c routing`` command.

.. rubric:: Run: ``SELECT * FROM osm_way_types ORDER BY type_id;``

.. code-block:: sql

     type_id |   name    
    ---------+-----------
           1 | highway
           2 | cycleway
           3 | tracktype
           4 | junction
    (4 rows)


.. rubric:: Run: ``SELECT * FROM osm_way_classes ORDER BY class_id;``

.. code-block:: sql

     class_id | type_id |       name        | priority | default_maxspeed 
    ----------+---------+-------------------+----------+------------------
          100 |       1 | road              |        1 |               50
          101 |       1 | motorway          |        1 |               50
          102 |       1 | motorway_link     |        1 |               50
          103 |       1 | motorway_junction |        1 |               50
          104 |       1 | trunk             |        1 |               50
          105 |       1 | trunk_link        |        1 |               50
          106 |       1 | primary           |        1 |               50
          107 |       1 | primary_link      |        1 |               50
          108 |       1 | secondary         |        1 |               50
          109 |       1 | tertiary          |        1 |               50
          110 |       1 | residential       |        1 |               50
          111 |       1 | living_street     |        1 |               50
          112 |       1 | service           |        1 |               50
          113 |       1 | track             |        1 |               50
          114 |       1 | pedestrian        |        1 |               50
          115 |       1 | services          |        1 |               50
          116 |       1 | bus_guideway      |        1 |               50
          117 |       1 | path              |        1 |               50
          118 |       1 | cycleway          |        1 |               50
          119 |       1 | footway           |        1 |               50
          120 |       1 | bridleway         |        1 |               50
          121 |       1 | byway             |        1 |               50
          122 |       1 | steps             |        1 |               50
          123 |       1 | unclassified      |        1 |               50
          124 |       1 | secondary_link    |        1 |               50
          125 |       1 | tertiary_link     |        1 |               50
          201 |       2 | lane              |        1 |               50
          202 |       2 | track             |        1 |               50
          203 |       2 | opposite_lane     |        1 |               50
          204 |       2 | opposite          |        1 |               50
          301 |       3 | grade1            |        1 |               50
          302 |       3 | grade2            |        1 |               50
          303 |       3 | grade3            |        1 |               50
          304 |       3 | grade4            |        1 |               50
          305 |       3 | grade5            |        1 |               50
          401 |       4 | roundabout        |        1 |               50
    (36 rows)


The road class is linked with the ways table by ``class_id`` field. After importing data the ``cost`` attribute is not set yet.
Its values can be changed with an ``UPDATE`` query.
In this example cost values for the classes table are assigned so that a circulating on faster roads is encouraged, so we execute:

.. code-block:: sql

    ALTER TABLE osm_way_classes ADD COLUMN penalty FLOAT;
    UPDATE osm_way_classes SET penalty=1;
    UPDATE osm_way_classes SET penalty=2.0 WHERE name IN ('pedestrian','steps','footway');
    UPDATE osm_way_classes SET penalty=1.5 WHERE name IN ('cicleway','living_street','path');
    UPDATE osm_way_classes SET penalty=0.8 WHERE name IN ('secondary','tertiary');
    UPDATE osm_way_classes SET penalty=0.6 WHERE name IN ('primary','primary_link');
    UPDATE osm_way_classes SET penalty=0.4 WHERE name IN ('trunk','trunk_link');
    UPDATE osm_way_classes SET penalty=0.3 WHERE name IN ('motorway','motorway_junction','motorway_link');

For better performance, especially if the network data is large, we are going to create an index on the ``class_id`` field of the `ways` table and `osm_way_classes` table. 

.. code-block:: sql

    CREATE INDEX  ON ways (class_id);
    CREATE INDEX  ON osm_way_classes (class_id);
    ALTER TABLE ways ADD CONSTRAINT class FOREIGN KEY (class_id) REFERENCES osm_way_classes (class_id);

The idea behind these two tables is to specify a factor to be multiplied with the cost of each link.


.. _p-8:

Exercise 8
........................................................

.. rubric:: Single Driver Routing encouraged to use faster roads.

* Driver “I am in vertex 30 and want to Drive to vertex 60 preferably on faster roads.”

.. rubric:: Problem description

* The driver wants to go from vertex 30 to vertex 60.
* The driver’s cost is in terms of length. In this case length is in degrees.
* osm2pgrouting cost and reverse_cost columns have lenght in degrees, but a negative length is used to indicate `wrong way`

.. rubric:: Query

.. code-block:: sql

    SELECT * FROM pgr_dijkstra('
        SELECT gid AS id,
            source,
            target,
            cost * penalty AS cost,
            reverse_cost * penalty AS reverse_cost
            FROM ways JOIN osm_way_classes 
            USING (class_id)',
        30, 60);

.. rubric:: Query Result

.. code-block:: sql

     seq | path_seq | node  | edge  |         cost         |       agg_cost       
    -----+----------+-------+-------+----------------------+----------------------
       1 |        1 |    30 | 50181 |  0.00063054853104263 |                    0
       2 |        2 | 13552 | 21550 | 0.000127435483677291 |  0.00063054853104263
       3 |        3 | 57785 | 21336 | 0.000123236695827811 | 0.000757984014719921
    ...
      61 |       61 | 60375 | 53879 |  0.00137580524785101 |   0.0323828545190476
      62 |       62 |    60 |    -1 |                    0 |   0.0337586597668986
    (62 rows)


.. _p-9:

Exercise 9
........................................................

.. rubric:: Restricted Access

* Driver “I am in vertex 30 and want to drive my lori to vertex 60 preferably on faster roads but I cant use walking roads and if I use primary road I have to pay a permit.”

.. rubric:: Problem description

* The driver wants to go from vertex 30 to vertex 60.
* The driver’s cost in this case will be in seconds.
* osm2pgrouting cost_s and reverse_cost_s columns,  but a negative ivalue is used to indicate `wrong way`
* Can not use `pedestrian`, `steps`, `footway`
* Big penatly if uses any kind of `primary`.


.. code-block:: sql

    UPDATE osm_way_classes SET penalty = 100 WHERE name LIKE 'primary%';

Through subqueries you can "mix" your costs as you like and this will change the results of your routing request immediately. Cost changes will affect the next shortest path search, and there is no need to rebuild your network.

Of course certain road classes can be excluded in the ``WHERE`` clause of the query as well, for example exclude "living_street" class:

.. rubric:: Query

.. code-block:: sql

    SELECT * FROM pgr_dijkstra('
        SELECT gid AS id,
            source,
            target,
            cost_s * penalty AS cost,
            reverse_cost_s * penalty AS reverse_cost
            FROM ways JOIN osm_way_classes 
            USING (class_id)
            WHERE class_id NOT IN (119,114,122)',
        30, 60);


.. rubric:: Query Result

.. code-block:: sql

     seq | path_seq | node  | edge  |       cost        |     agg_cost     
    -----+----------+-------+-------+-------------------+------------------
       1 |        1 |    30 | 59650 |  1.84464624769373 |                0
       2 |        2 | 22440 | 64869 |  18.2593026761421 | 1.84464624769373
       3 |        3 | 15707 | 67254 |  5.07629932393229 | 20.1039489238358
    ...
      56 |       56 | 26872 |  7771 |  7.22278723655835 | 370.760121671705
      57 |       57 |    60 |    -1 |                 0 | 377.982908908263
    (57 rows)

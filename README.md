# highways-for-pgrouting
Building OS Highways for pgRouting

## 1. Load the data

If you have Safe Software’s FME you can download the workbenches to create the database tables here: https://github.com/OrdnanceSurvey/OSMM-Highways-Network-Support but note that you will have to edit the workbenches to write to PostGIS instead of GeoPackage.  Not hard if you’ve used FME before.

The following tables are created by FME process:

- hw_accessrestriction*
- hw_accessrestriction_inclusion
- hw_accessrestriction_exemption
- hw_accessrestriction_networkref*
- hw_restrictionforvehicles
- hw_restrictionsforvehicles_networkref
- hw_road
- hw_road_networkref
- hw_roadlink*
- hw_roadlink_formspartof
- hw_roadlink_relatedroadarea
- hw_roadnode*
- hw_roadnode_relatedroadarea
- hw_turnrestriction*
- hw_turnrestriction_networkref*

\* Tables used in this exercise

## 2. Add the fields required for pgRouting

        ALTER TABLE my_schema.hw_roadlink
          ADD COLUMN source integer,
          ADD COLUMN target integer,
          ADD COLUMN one_way varchar(2),
          ADD COLUMN speed_km integer,
          ADD COLUMN cost_len double precision,
          ADD COLUMN rcost_len double precision,
          ADD COLUMN cost_time double precision,
          ADD COLUMN rcost_time double precision,
          ADD COLUMN x1 double precision,
          ADD COLUMN y1 double precision,
          ADD COLUMN x2 double precision,
          ADD COLUMN y2 double precision,
          ADD COLUMN to_cost double precision,
          ADD COLUMN rule text,
          ADD COLUMN isolated integer;

## 3. Create indexes for speed

        CREATE INDEX hw_roadlink_source_idx ON my_schema.hw_roadlink USING btree(source);
        CREATE INDEX hw_roadlink_target_idx ON my_schema.hw_roadlink USING btree(target);

## 4. Update new fields with values required for pgRouting

The start and end coordinates of the links are used in the Astar shortest path algorithmto determine the links closest to the target shortest path solution.

        UPDATE my_schema.hw_roadlink 
          SET x1 = st_x(st_startpoint(centrelinegeometry)),
            y1 = st_y(st_startpoint(centrelinegeometry)),
            x2 = st_x(st_endpoint(centrelinegeometry)),
            y2 = st_y(st_endpoint(centrelinegeometry));

Setting the one_way flags helps pgRouting analyse the road network graph for errors.

        UPDATE hw_roadlink SET one_way = 'B' WHERE directionality = 'bothDirections';
        UPDATE hw_roadlink SET one_way = 'TF' WHERE directionality = 'inOppositeDirection';
        UPDATE hw_roadlink SET one_way = 'FT' WHERE directionality = 'inDirection';

Costs are what the routing engine uses to calculate the best route across the network.  In this example we are using **distance**, based on link length, and **time**, based on average speed for particular road classification and link length.  For links with one way directionality we set the reverse cost very high to discourage use.  pgRouting uses the digitised direction of the line to help with routing and the Highways layer helpfully has a field which tells us, for one way streets, which way the line has been drawn and which way the traffic is expected to flow.  For one way streets with traffic flow in the same direction as digitised direction the directionality flag is set to **inDirection**. For traffic flow contra to line direction it is set to **inOppositeDirection**. For links with two way flow of traffic it is set to **bothDirections**.

Forward and reverse cost is the same for two way streets

        UPDATE my_schema.hw_roadlink
          SET cost_len = ST_Length(centrelinegeometry),
              rcost_len = ST_Length(centrelinegeometry)
          WHERE directionality IN ('bothDirections');

Reverse costs are increased for one way streets

        UPDATE my_schema.hw_roadlink
          SET cost_len = ST_Length(centrelinegeometry),
              rcost_len = ST_Length(centrelinegeometry)*100
          WHERE directionality IN ('inDirection');
        
        UPDATE my_schema.hw_roadlink
          SET rcost_len = ST_Length(centrelinegeometry),
              cost_len = ST_Length(centrelinegeometry)*100
          WHERE directionality IN ('inOppositeDirection');
          
To set the time cost I set an average speed based on the road classifiation and form of the road.  A tip here is to remember to set the speed for slip roads, traffic island links to be the same as the speed for the carriageway of the same class.  This prevents odd routing through complex junctions.

Check what type of roads you are using with:

        SELECT DISTINCT roadclassification, formofway
          FROM my_schema.hw_roadlink
          ORDER BY roadclassification;
          
Set the road speed by classification and form. Adjust as required

        UPDATE my_schema.hw_roadlink
        SET speed_km = 
        CASE
        	WHEN roadclassification = 'A Road' AND formofway = 'Roundabout' THEN 40
        	WHEN roadclassification = 'A Road' AND formofway = 'Dual Carriageway' THEN 100
        	WHEN roadclassification = 'A Road' AND formofway = 'Traffic Island Link' THEN 100
        	WHEN roadclassification = 'A Road' AND formofway = 'Slip Road' THEN 100
        	WHEN roadclassification = 'A Road' AND formofway = 'Traffic Island Link At Junction' THEN 100
        	WHEN roadclassification = 'A Road' AND formofway = 'Single Carriageway' THEN 100
        	WHEN roadclassification = 'B Road' AND formofway = 'Single Carriageway' THEN 80
        	WHEN roadclassification = 'B Road' AND formofway = 'Slip Road' THEN 80
        	WHEN roadclassification = 'B Road' AND formofway = 'Roundabout' THEN 40
        	WHEN roadclassification = 'B Road' AND formofway = 'Traffic Island Link At Junction' THEN 80
        	WHEN roadclassification = 'Not Classified' AND formofway = 'Traffic Island Link' THEN 60
        	WHEN roadclassification = 'Not Classified' AND formofway = 'Single Carriageway' THEN 60
        	WHEN roadclassification = 'Not Classified' AND formofway = 'Roundabout' THEN 40
        	WHEN roadclassification = 'Not Classified' AND formofway = 'Dual Carriageway' THEN 100
        	WHEN roadclassification = 'Not Classified' AND formofway = 'Enclosed Traffic Area' THEN 40 
        	WHEN roadclassification = 'Not Classified' AND formofway = 'Traffic Island Link At Junction' THEN 60
        	WHEN roadclassification = 'Unclassified' AND formofway = 'Slip Road' THEN 40
        	WHEN roadclassification = 'Unclassified' AND formofway = 'Enclosed Traffic Area' THEN 40
        	WHEN roadclassification = 'Unclassified' AND formofway = 'Single Carriageway' THEN 40
        	WHEN roadclassification = 'Unclassified' AND formofway = 'Layby' THEN 10
        	WHEN roadclassification = 'Unclassified' AND formofway = 'Traffic Island Link At Junction' THEN 40
        	WHEN roadclassification = 'Unclassified' AND formofway = 'Traffic Island Link' THEN 40
        	WHEN roadclassification = 'Unclassified' AND formofway = 'Dual Carriageway' THEN 100
        	WHEN roadclassification = 'Unclassified' AND formofway = 'Roundabout' THEN 40
        	ELSE 1
        END;
        
Update the travel time costs for two way roads.

        UPDATE my_schema.hw_roadlink
          SET cost_time = ST_Length(centrelinegeometry)/1000.0/speed_km::numeric*3600.0,
              rcost_time = ST_Length(centrelinegeometry)/1000.0/speed_km::numeric*3600.0
          WHERE directionality IN ('bothDirections');

Update the travel time costs for one way streets digitised the same way as traffic flow.

        UPDATE my_schema.hw_roadlink
          SET cost_time = ST_Length(centrelinegeometry)/1000.0/speed_km::numeric*3600.0,
              rcost_time = ST_Length(centrelinegeometry)*100/1000.0/speed_km::numeric*3600.0
          WHERE directionality IN ('inDirection');

Set the travel time costs for one way streets digitised against the traffic flow.

        UPDATE my_schema.hw_roadlink
          SET rcost_time = ST_Length(centrelinegeometry)/1000.0/speed_km::numeric*3600.0,
              cost_time = ST_Length(centrelinegeometry)*100/1000.0/speed_km::numeric*3600.0
          WHERE directionality IN ('inOppositeDirection');
          
## 5. Initial build of routing network

Build network topology

        SELECT pgr_createTopology('my_schema.hw_roadlink', 0.001, 'centrelinegeometry', 'ogc_fid', 'source', 'target');

Check topology for errors

        SELECT pgr_analyzeGraph('my_schema.hw_roadlink', 0.001, ' centrelinegeometry ', 'ogc_fid', 'source', 'target'); 

Check for one-way errors

        SELECT pgr_analyzeOneway('my_schema.hw_roadlink',
            ARRAY['', 'B', 'TF'],
            ARRAY['', 'B', 'FT'],
            ARRAY['', 'B', 'FT'],
            ARRAY['', 'B', 'TF'],
            oneway:='one_way'
            );

Identify links with problems

        SELECT * FROM hw_roadlink_vertices_pgr WHERE chk = 1;

Identify links with deadends
        
        SELECT * FROM hw_roadlink_vertices_pgr WHERE cnt = 1;

Identify isolated segments

        SELECT * FROM hw_roadlink a, hw_roadlink_vertices_pgr b, hw_roadlink_vertices_pgr c 
        WHERE a.source = b.id AND b.cnt = 1 AND a.target = c.id AND c.cnt = 1;

Identify nodes with problems

        SELECT * FROM hw_roadlink_vertices_pgr WHERE ein = 0 OR eout = 0;

Identify the links at nodes with problems

        SELECT gid FROM hw_roadlink a, hw_roadlink_vertices_pgr b WHERE a.source=b.id AND ein=0 OR eout=0
          UNION
        SELECT gid FROM hw_roadlink a, hw_roadlink_vertices_pgr b WHERE a.target=b.id AND ein=0 OR eout=0;

## 6. Adding in turn restrictions

### “No Turn” turn restrictions

The Highways dataset provides tables of turn restrictions with a reference to the road links, the type of restriction and a turn sequence number.  The turn restriction table is joined to the road link table and is used to generate the turn restrictions in pgRouting format.

First, we create a view of the links involved in the turn restriction. We select the FROM link with a sequence of 0 and the TO link with a sequence number of 1 where the restriction type is NO TURN.

        CREATE OR REPLACE VIEW view_hw_nt_links AS
        SELECT row_number() OVER () AS gid, 
        	a.roadlink_toid AS from_link, 
        	c.ogc_fid AS from_fid, 
        	b.roadlink_toid AS to_link, 
        	d.ogc_fid AS to_fid, 
        	a.applicabledirection, 
        	a.sequence AS from_seq, 
        	b.sequence AS to_seq, 
        	a.restriction, 
        	c.centrelinegeometry
        FROM hw_turnrestriction_networkref a
        INNER JOIN hw_turnrestriction_networkref b ON a.toid = b.toid 
        INNER JOIN hw_roadlink c ON c.toid = a.roadlink_toid
        INNER JOIN hw_roadlink d ON d.toid = b.roadlink_toid
        WHERE b.sequence = 1 
        AND a.sequence = 0
        AND a.restriction = 'No Turn';

### “Mandatory Turn” turn restrictions

Mandatory turns are modelled in the dataset as turns which are allowed. So there is the entry link and the exit link which, together with the associated node make up the mandatory turn.  To create a restriction here to enforce the mandatory turn you need to select all the other links in the turn and restrict access to them so that the only way is Essex.

Create a view of the entry link and associated node

        CREATE OR REPLACE VIEW view_hw_mt_link1 AS
        SELECT a.roadlink_toid,
          b.ogc_fid, 
          a.sequence,
          a.applicabledirection, 
          (CASE WHEN a.applicabledirection = 'inOppositeDirection' AND sequence = 0 THEN b.startnode 
            WHEN a.applicabledirection = 'inDirection' AND sequence = 1 THEN b.startnode
            ELSE b.endnode END) AS nodetoid,
          b.centrelinegeometry 
        FROM hw_turnrestriction_networkref a
        INNER JOIN hw_roadlink b ON a.roadlink_toid = b.toid
        WHERE a.restriction = 'Mandatory Turn'
        AND a.sequence = 0;
        
Then create a view of the exit link.

        CREATE OR REPLACE VIEW view_hw_mt_link2 AS
        SELECT a.roadlink_toid,
          b.ogc_fid, 
          a.sequence,
          a.applicabledirection, 
          (CASE WHEN a.applicabledirection = 'inOppositeDirection' AND sequence = 0 THEN b.startnode 
            WHEN a.applicabledirection = 'inDirection' AND sequence = 1 THEN b.startnode
            ELSE b.endnode END) AS nodetoid,
          b.centrelinegeometry  
        FROM hw_turnrestriction_networkref a
        INNER JOIN hw_roadlink b ON a.roadlink_toid = b.toid
        WHERE restriction = 'Mandatory Turn'
        AND sequence = 1;
        
Combine these to select the other links in the turn, the links you cannot turn into.

        CREATE OR REPLACE VIEW view_hw_mt_nt_links AS
        SELECT row_number() OVER () AS gid,
          a.roadlink_toid AS from_toid,
          a.ogc_fid AS from_fid,
          b.toid AS to_toid,
          b.ogc_fid AS to_fid
        FROM view_hw_mt_link1 a
        INNER JOIN hw_roadlink b ON (a.nodetoid = b.startnode OR a.nodetoid = b.endnode)
        AND a.roadlink_toid <> b.toid
        AND b.toid NOT IN (SELECT roadlink_toid FROM view_hw_mt_link2);
        

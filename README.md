# NotesProject
Repository to generate notes for content based on player metrics from vendors.

## Overview
So far this repository has mapped out the back end for generating the notes. This happens in 2 phases. The first phase is in AWS Athena which creates 2 views that aggregate all the data needed to be components of notes such as event_id, player_id, max fastball velo, event rank of fastball velo, national percentile rank of fastball velo, etc. See the SQL section of the code below for a more expanded explanation. The second part of the back end so far is the python way of sifting through a view's data and pulling out what players qualify for a notes. Python does this through a rules engine that loops through the view and appends to a running dataframe that holds all of the notes. 

## Note Ideas
A google sheet has been created where various people have been inserting various ideas of notes. This is the breeding ground for notes ideas and what I've been building the notes on up to this point. Note that what has been created does not implement all of the notes ideas in the google sheet (see errors/shortcomings section below), but probably knocks out 90% of the note ideas in the google sheet.

The sheet is at this link: https://docs.google.com/spreadsheets/d/1FL0iF3S-lJuYOWeEc-wxZp0ix72fEIbMccef27UJSDo/edit#gid=1195031060

## SQL
The SQL portion of the code is split into 2 views that currently (as of 2/15/2022) reside in the ```datamart_dev``` database in Athena. Each view tries to pull all necessary metrics that will be used in the parameters of the notes later. Currently, each view has the event_id, player_id, actual metrics (fastball velo, exit speed, 60 time, etc.), ranking by event (rotational acceleration event ranking, curveball spin event ranking, etc), and finally, the national percentile, compared to all players in our database (exit speed national percentile, hop national percentile, etc.) The code for the views is as follows:

```datamart_dev.notes_batter_rankings```:
``` 
    SELECT
     a.event_id
   , a.player_id
   , a.avg_exitspeed
   , a.max_exitspeed
   , a.avg_distance
   , a.max_distance
   , a.hard_ball_percentage
   , a.sweetspot_percentage
   , a.avg_bat_speed
   , a.max_bat_speed
   , a.avg_peak_hand_speed
   , a.max_peak_hand_speed
   , a.avg_rotational_acceleration
   , a.max_rotational_acceleration
   , a.avg_on_plane_efficiency
   , b."60 time" sixty
   , b."pop time low" pop_time_min
   , b."position velocity- catcher" catcher_velo
   , b."position velocity- infield" infield_velo
   , b."position velocity- outfield" outfield_velo
   , (CASE WHEN (b."60 time" IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY b.event_id ORDER BY (CASE WHEN (b."pop time low" IS NULL) THEN 0 ELSE b."pop time low" END) ASC) END) event_pop_time_rank
   , (CASE WHEN (b."60 time" IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY b.event_id ORDER BY (CASE WHEN (b."60 time" IS NULL) THEN 0 ELSE b."60 time" END) ASC) END) event_sixty_rank
   , (CASE WHEN (b."position velocity- catcher" IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY b.event_id ORDER BY (CASE WHEN (b."position velocity- catcher" IS NULL) THEN 0 ELSE b."position velocity- catcher" END) DESC) END) event_catcher_velo_rank
   , (CASE WHEN (b."position velocity- infield" IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY b.event_id ORDER BY (CASE WHEN (b."position velocity- infield" IS NULL) THEN 0 ELSE b."position velocity- infield" END) DESC) END) event_infield_velo_rank
   , (CASE WHEN (b."position velocity- outfield" IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY b.event_id ORDER BY (CASE WHEN (b."position velocity- outfield" IS NULL) THEN 0 ELSE b."position velocity- outfield" END) DESC) END) event_outfield_velo_rank
   , (CASE WHEN (a.avg_exitspeed IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id ORDER BY (CASE WHEN (a.avg_exitspeed IS NULL) THEN 0 ELSE a.avg_exitspeed END) DESC) END) event_avg_exitspeed_rank
   , (CASE WHEN (a.max_exitspeed IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id ORDER BY (CASE WHEN (a.max_exitspeed IS NULL) THEN 0 ELSE a.max_exitspeed END) DESC) END) event_max_exitspeed_rank
   , (CASE WHEN (a.avg_distance IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id ORDER BY (CASE WHEN (a.avg_distance IS NULL) THEN 0 ELSE a.avg_distance END) DESC) END) event_avg_distance_rank
   , (CASE WHEN (a.max_distance IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id ORDER BY (CASE WHEN (a.max_distance IS NULL) THEN 0 ELSE a.max_distance END) DESC) END) event_max_distance_rank
   , (CASE WHEN (a.hard_ball_percentage IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id ORDER BY (CASE WHEN (a.hard_ball_percentage IS NULL) THEN 0 ELSE a.hard_ball_percentage END) DESC) END) event_avg_hard_ball_percentage_rank
   , (CASE WHEN (a.sweetspot_percentage IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id ORDER BY (CASE WHEN (a.sweetspot_percentage IS NULL) THEN 0 ELSE a.sweetspot_percentage END) DESC) END) event_sweetspot_percentage_rank
   , (CASE WHEN (a.avg_bat_speed IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id ORDER BY (CASE WHEN (a.avg_bat_speed IS NULL) THEN 0 ELSE a.avg_bat_speed END) DESC) END) event_avg_bat_speed_rank
   , (CASE WHEN (a.max_bat_speed IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id ORDER BY (CASE WHEN (a.max_bat_speed IS NULL) THEN 0 ELSE a.max_bat_speed END) DESC) END) event_max_bat_speed_rank
   , (CASE WHEN (a.avg_peak_hand_speed IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id ORDER BY (CASE WHEN (a.avg_peak_hand_speed IS NULL) THEN 0 ELSE a.avg_peak_hand_speed END) DESC) END) event_avg_peak_hand_speed_rank
   , (CASE WHEN (a.max_peak_hand_speed IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id ORDER BY (CASE WHEN (a.max_peak_hand_speed IS NULL) THEN 0 ELSE a.max_peak_hand_speed END) DESC) END) event_max_peak_hand_speed_rank
   , (CASE WHEN (a.avg_rotational_acceleration IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id ORDER BY (CASE WHEN (a.avg_rotational_acceleration IS NULL) THEN 0 ELSE a.avg_rotational_acceleration END) DESC) END) event_avg_rotational_acceleration_rank
   , (CASE WHEN (a.max_rotational_acceleration IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id ORDER BY (CASE WHEN (a.max_rotational_acceleration IS NULL) THEN 0 ELSE a.max_rotational_acceleration END) DESC) END) event_max_rotational_acceleration_rank
   , (CASE WHEN (a.avg_on_plane_efficiency IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id ORDER BY (CASE WHEN (a.avg_on_plane_efficiency IS NULL) THEN 0 ELSE a.avg_on_plane_efficiency END) DESC) END) event_avg_on_plane_efficiency_rank
   , (CASE WHEN (a.avg_exitspeed IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (a.avg_exitspeed IS NULL) THEN 0 ELSE a.avg_exitspeed END) ASC) END) avg_exitspeed_overall_percentile
   , (CASE WHEN (a.max_exitspeed IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (a.max_exitspeed IS NULL) THEN 0 ELSE a.max_exitspeed END) ASC) END) max_exitspeed_overall_percentile
   , (CASE WHEN (a.hard_ball_percentage IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (a.hard_ball_percentage IS NULL) THEN 0 ELSE a.hard_ball_percentage END) ASC) END) hard_ball_percentage_overall_percentile
   , (CASE WHEN (a.sweetspot_percentage IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (a.sweetspot_percentage IS NULL) THEN 0 ELSE a.sweetspot_percentage END) ASC) END) sweetspot_percentage_overall_percentile
   , (CASE WHEN (a.avg_bat_speed IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (a.avg_bat_speed IS NULL) THEN 0 ELSE a.avg_bat_speed END) ASC) END) avg_bat_speed_overall_percentile
   , (CASE WHEN (a.max_bat_speed IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (a.max_bat_speed IS NULL) THEN 0 ELSE a.max_bat_speed END) ASC) END) max_bat_speed_overall_percentile
   , (CASE WHEN (a.avg_peak_hand_speed IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (a.avg_peak_hand_speed IS NULL) THEN 0 ELSE a.avg_peak_hand_speed END) ASC) END) avg_peak_hand_speed_overall_percentile
   , (CASE WHEN (a.max_peak_hand_speed IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (a.max_peak_hand_speed IS NULL) THEN 0 ELSE a.max_peak_hand_speed END) ASC) END) max_peak_hand_speed_overall_percentile
   , (CASE WHEN (a.avg_rotational_acceleration IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (a.avg_rotational_acceleration IS NULL) THEN 0 ELSE a.avg_rotational_acceleration END) ASC) END) avg_rotational_acceleration_overall_percentile
   , (CASE WHEN (a.max_rotational_acceleration IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (a.max_rotational_acceleration IS NULL) THEN 0 ELSE a.max_rotational_acceleration END) ASC) END) max_rotational_acceleration_overall_percentile
   , (CASE WHEN (a.avg_on_plane_efficiency IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (a.avg_on_plane_efficiency IS NULL) THEN 0 ELSE a.avg_on_plane_efficiency END) ASC) END) avg_on_plane_efficiency_overall_percentile
   , (CASE WHEN (b."60 time" IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (b."60 time" IS NULL) THEN 0 ELSE b."60 time" END) DESC) END) sixty_overall_percentile
   , (CASE WHEN (b."position velocity- catcher" IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (b."position velocity- catcher" IS NULL) THEN 0 ELSE b."position velocity- catcher" END) ASC) END) catcher_velo_overall_percentile
   , (CASE WHEN (b."position velocity- infield" IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (b."position velocity- infield" IS NULL) THEN 0 ELSE b."position velocity- infield" END) ASC) END) infield_velo_overall_percentile
   , (CASE WHEN (b."position velocity- outfield" IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (b."position velocity- outfield" IS NULL) THEN 0 ELSE b."position velocity- outfield" END) ASC) END) outfield_velo_overall_percentile
   , (CASE WHEN (b."pop time low" IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (b."pop time low" IS NULL) THEN 0 ELSE b."pop time low" END) DESC) END) pop_time_overall_percentile
   FROM
     ("datamart"."pbr_event_batter_metrics" a
   FULL JOIN "datawarehouse"."player_event_stats_pivot" b ON ((a.player_id = b.pbr_id) AND (a.event_id = b.event_id)))
   ```
   
   ```datamart_dev.notes_batter_rankings```:
   ```
   SELECT
  a.event_id
, a.player_id
, a.taggedpitchtype
, a.total_pitches
, a.avg_effvelocity
, a.max_relspeed
, a.avg_spinrate
, a.peak_spinrate
, a.avg_inducedvertbreak
, a.peak_inducedvertbreak
, a.avg_horzbreak
, a.peak_horzbreak
, (a.avg_inducedvertbreak + a.avg_horzbreak) avg_total_break
, (a.peak_inducedvertbreak + a.peak_horzbreak) peak_total_break
, b.hop_plus
, b.sink_plus
, b.rise_plus
, b.hammer_plus
, b.sweep_plus
, (CASE WHEN (a.max_relspeed IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id, a.taggedpitchtype ORDER BY (CASE WHEN (a.max_relspeed IS NULL) THEN 0 ELSE a.max_relspeed END) DESC) END) event_max_velo_rank
, (CASE WHEN (a.avg_effvelocity IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id, a.taggedpitchtype ORDER BY (CASE WHEN (a.avg_effvelocity IS NULL) THEN 0 ELSE a.avg_effvelocity END) DESC) END) event_avg_velo_rank
, (CASE WHEN (a.avg_spinrate IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id, a.taggedpitchtype ORDER BY (CASE WHEN (a.avg_spinrate IS NULL) THEN 0 ELSE a.avg_spinrate END) DESC) END) event_avg_spin_rank
, (CASE WHEN (a.peak_spinrate IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id, a.taggedpitchtype ORDER BY (CASE WHEN (a.peak_spinrate IS NULL) THEN 0 ELSE a.peak_spinrate END) DESC) END) event_peak_spinrate_rank
, (CASE WHEN (a.avg_inducedvertbreak IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id, a.taggedpitchtype ORDER BY (CASE WHEN (a.avg_inducedvertbreak IS NULL) THEN 0 ELSE a.avg_inducedvertbreak END) DESC) END) event_avg_inducedvertbreak_rank
, (CASE WHEN (a.peak_inducedvertbreak IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id, a.taggedpitchtype ORDER BY (CASE WHEN (a.peak_inducedvertbreak IS NULL) THEN 0 ELSE a.peak_inducedvertbreak END) DESC) END) event_max_inducedvertbreak_rank
, (CASE WHEN (a.avg_horzbreak IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id, a.taggedpitchtype ORDER BY (CASE WHEN (a.avg_horzbreak IS NULL) THEN 0 ELSE a.avg_horzbreak END) DESC) END) event_avg_horzbreak_rank
, (CASE WHEN (a.peak_horzbreak IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id, a.taggedpitchtype ORDER BY (CASE WHEN (a.peak_horzbreak IS NULL) THEN 0 ELSE a.peak_horzbreak END) DESC) END) event_max_horzbreak_rank
, (CASE WHEN ((a.avg_inducedvertbreak IS NULL) OR (a.avg_horzbreak IS NULL)) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id, a.taggedpitchtype ORDER BY (CASE WHEN ((a.avg_horzbreak IS NULL) OR (a.avg_inducedvertbreak IS NULL)) THEN 0 ELSE (a.avg_horzbreak + a.avg_inducedvertbreak) END) DESC) END) event_avg_total_break_rank
, (CASE WHEN ((a.peak_inducedvertbreak IS NULL) OR (a.peak_horzbreak IS NULL)) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id, a.taggedpitchtype ORDER BY (CASE WHEN ((a.peak_horzbreak IS NULL) OR (a.peak_inducedvertbreak IS NULL)) THEN 0 ELSE (a.peak_horzbreak + a.peak_inducedvertbreak) END) DESC) END) event_peak_total_break_rank
, (CASE WHEN (b.hop_plus IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id, a.taggedpitchtype ORDER BY (CASE WHEN (b.hop_plus IS NULL) THEN 0 ELSE b.hop_plus END) DESC) END) event_hop_plus_rank
, (CASE WHEN (b.sink_plus IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id, a.taggedpitchtype ORDER BY (CASE WHEN (b.sink_plus IS NULL) THEN 0 ELSE b.sink_plus END) DESC) END) event_sink_plus_rank
, (CASE WHEN (b.rise_plus IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id, a.taggedpitchtype ORDER BY (CASE WHEN (b.rise_plus IS NULL) THEN 0 ELSE b.rise_plus END) DESC) END) event_rise_plus_rank
, (CASE WHEN (b.hammer_plus IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id, a.taggedpitchtype ORDER BY (CASE WHEN (b.hammer_plus IS NULL) THEN 0 ELSE b.hammer_plus END) DESC) END) event_hammer_plus_rank
, (CASE WHEN (b.sweep_plus IS NULL) THEN null ELSE "rank"() OVER (PARTITION BY a.event_id, a.taggedpitchtype ORDER BY (CASE WHEN (b.sweep_plus IS NULL) THEN 0 ELSE b.sweep_plus END) DESC) END) event_sweep_plus_rank
, (CASE WHEN (a.max_relspeed IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (a.max_relspeed IS NULL) THEN 0 ELSE a.max_relspeed END) ASC) END) max_relspeed_overall_percentile
, (CASE WHEN (a.avg_effvelocity IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (a.avg_effvelocity IS NULL) THEN 0 ELSE a.avg_effvelocity END) ASC) END) avg_relspeed_overall_percentile
, (CASE WHEN (a.avg_spinrate IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (a.avg_spinrate IS NULL) THEN 0 ELSE a.avg_spinrate END) ASC) END) avg_spinrate_overall_percentile
, (CASE WHEN (a.peak_spinrate IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (a.peak_spinrate IS NULL) THEN 0 ELSE a.peak_spinrate END) ASC) END) peak_spinrate_overall_percentile
, (CASE WHEN (a.avg_inducedvertbreak IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (a.avg_inducedvertbreak IS NULL) THEN 0 ELSE a.avg_inducedvertbreak END) ASC) END) avg_inducedvertbreak_overall_percentile
, (CASE WHEN (a.peak_inducedvertbreak IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (a.peak_inducedvertbreak IS NULL) THEN 0 ELSE a.peak_inducedvertbreak END) ASC) END) peak_inducedvertbreak_overall_percentile
, (CASE WHEN (a.avg_horzbreak IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (a.avg_horzbreak IS NULL) THEN 0 ELSE a.avg_horzbreak END) ASC) END) avg_horzbreak_overall_percentile
, (CASE WHEN (a.peak_horzbreak IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (a.peak_horzbreak IS NULL) THEN 0 ELSE a.peak_horzbreak END) ASC) END) peak_horzbreak_overall_percentile
, (CASE WHEN ((a.avg_inducedvertbreak IS NULL) OR (a.avg_horzbreak IS NULL)) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN ((a.avg_horzbreak IS NULL) OR (a.avg_inducedvertbreak IS NULL)) THEN 0 ELSE (a.avg_horzbreak + a.avg_inducedvertbreak) END) DESC) END) avg_total_break_overall_percentile
, (CASE WHEN ((a.peak_inducedvertbreak IS NULL) OR (a.peak_horzbreak IS NULL)) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN ((a.peak_horzbreak IS NULL) OR (a.peak_inducedvertbreak IS NULL)) THEN 0 ELSE (a.peak_horzbreak + a.peak_inducedvertbreak) END) DESC) END) peak_total_break_rank_overall_percentile
, (CASE WHEN (b.hop_plus IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (b.hop_plus IS NULL) THEN 0 ELSE b.hop_plus END) ASC) END) hop_plus_overall_percentile
, (CASE WHEN (b.sink_plus IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (b.sink_plus IS NULL) THEN 0 ELSE b.sink_plus END) ASC) END) sink_plus_overall_percentile
, (CASE WHEN (b.rise_plus IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (b.rise_plus IS NULL) THEN 0 ELSE b.rise_plus END) ASC) END) rise_plus_overall_percentile
, (CASE WHEN (b.hammer_plus IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (b.hammer_plus IS NULL) THEN 0 ELSE b.hammer_plus END) ASC) END) hammer_plus_overall_percentile
, (CASE WHEN (b.sweep_plus IS NULL) THEN null ELSE "percent_rank"() OVER (ORDER BY (CASE WHEN (b.sweep_plus IS NULL) THEN 0 ELSE b.sweep_plus END) ASC) END) sweep_plus_overall_percentile
FROM
  ("datamart_dev"."pbr_event_player_metrics" a
LEFT JOIN "datamart_dev"."pbr_event_pitch_score_metrics" b ON (((a.player_id = b.player_id) AND (a.event_id = b.event_id)) AND (a.taggedpitchtype = b.taggedpitchtype))
```
A couple things to note: 
- It is worth noting the ascending vs. descending is important to keep in mind whenever doing the event rankings and the percentile rankings. In most cases we want the event ranking to be DESCENDING since we want the "best" ranking to be 1 (the smallest number). However, for percentiles, it's the opposite, we want the rankings to be ASCENDING because a percentile of 0.01 is the "worst" while a percentile of 0.99 is the "best". Also note the ascending/descending rules flip for 60 times where the "best" 60 times and pop times where the "best" pop time is the _smallest_ value, not the largest one as it is with other metrics.
- Also, dealing with nulls is very important here. Whenever you're ranking SQL ranks a null values as infinity, without ignoring nulls, if a player had a null exit velo and another player had an exit velo of 120, the player with the null exit velo would be ranked first. I dealt with this in the code above by checking if the value is null, if it is, then skipping the ranking and assigning the field as null for the record. We need to do similarly for the partition argument. We need to "filter out" all null values from percentile and ranking calculation. We do this in the partition argument by adding a case statement, to include the value whenever it's not null (assign it 1), but exclude whenever it's null (assign it 0).

## Python
The role of the python script is to take the values calculated in the SQL views and based on the filtering conditions given in the google sheet, find the records that qualify for a note and return a dataframe representing all of the data parameters we need to pass into a note sentence. The python code does this by first reading in the data and assigning an empty dataframe to hold the notes. Then, python uses a rules engine to loop through all of the notes rules we have so far. Whenever a condition is met for a note, the values are appended to the dataframe and the loop continues until the data from SQL has been completely read through.

## Errors/Shortcomings
- We do not have some metrics that we need on the notes. Many of the notes require averages for D1, Draft League, MLB, etc. We do not have these numbers yet. 
- As of now, the design is very rigid which will lead to problems as we expand the notes further. 
- Whenever we want to add new metrics or filteres to a notes, we need to keep adding to the SQL queries which are already very large.
- We do not have a query to "cross over" batting and pitching. We can generate pitcher notes and we can generate batter notes, but we cannot do a note like "Joe had a maximum fastball velo of 98 that ranked first at event and an average exit velo of 113 that ranked first at event. We need to re-factor in a way that crosses over both batting and pitching.
- The rules engine is very slow as it loops through each record of the read in data one by one. This isn't a huge deal if we run the engine every day, but if we try to run the engine once every month, or even once every week, the lambda execution time will skyrocked. 
- The rules engine doesn't deal with null values well. Some notes require that we compare two metrics and take the maximum or minimum. For example, we want a note if a breaking ball spin is over 3,000. Consider a player that has a curveball spin of 3050, but did not through a slider. In the code, we want to take the maximum of the two pitche's spin. However, this fails because max(curveball_spin,slider_spin) returns NaN. I tried using numpy, but that also failed as the rules engine gave me a NotImplementedError.

## Next Steps
- Re-design system to fix or correct the errors/shortcomings above.
- Meet with scouts and coaches to get more ideas of notes and more wording on how to phrase notes.
- Find a way to say a note in multiple ways.
- Go from taking what we've done here (pulling out values that qualify for a note) to actually generating the entire sentence.

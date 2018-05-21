## Attribution

Map tiles from http://maps.stamen.com/.

## Data extraction

1. download eligibility data from http://overpass-turbo.eu/s/vq5
as export.geojson

2. download gym data from https://gymhuntr.com, using Chrome, by opening
the devtools (specifically the Network tab), requesting EVERY BIT of the
city (make sure you cover it all). Then right click on a request, and
save all + content as HAR (gymhuntr.har).

3. from gym data, extract relevant gyms using PHP:

    ```php
    <?php
    $data = json_decode(file_get_contents("gymhuntr.har"));
    $URL_REGEXP = '!^https://api.gymhuntr.com/api/gyms\?latitude=([0-9.]+)&longitude=([0-9.]+)&!';

    foreach ($data->log->entries as $e)
    {
        if (!preg_match($URL_REGEXP, $e->request->url, $m))
        {
            continue;
        }
        
        printf("URL: %s, lat=%f, long=%f\n", $e->request->url, $m[1], $m[2]);
        
        $data_inside = json_decode($e->response->content->text);
        /** to simply dump the JSONs from the gyms **
        file_put_contents("gymhuntr_".$m[1]."_".$m[2].".json", implode("\n", $data_inside->gyms));
        */
        $data_csv = '';
        foreach ($data_inside->gyms as $g)
        {
            $data_gym = json_decode($g);
            $data_csv .= sprintf("%s,%f,%f\n",
                                 trim(strtr($data_gym->gym_name, '",', '  ')),
                                 $data_gym->longitude,
                                 $data_gym->latitude);
        }
        file_put_contents("gymhuntr_".$m[1]."_".$m[2].".json", $data_csv."\n");
    }
    ?>
    ```

4. now make one big deduplicated CSV from all those smaller ones:

    ```bash
    cat gymhuntr_48.* | sort -u > uniq_gymhuntr.csv
    ```

5. download osmcoverer_with_centers from
https://github.com/MzHub/osmcoverer/releases/tag/v2.1.2
and execute :

    ```bash
    WINEPREFIX=~/.wine64 wine ./osmcoverer_with_centers.exe -checkcellcenters -markers uniq_gymhuntr.csv export.geojson
    cp output/cellcenters_within_features.csv eligible_gyms.csv
    ```

6. (opt.) add sponsored gyms to eligible_gyms.csv. They can be found at:
https://www.pokemongomap.info/location/48,858736/2,354499/13
CSV format: Gym name,long,lat

7. generate a geojson with gyms + level --12-- 13 S2 cells

    ```
    WINEPREFIX=~/.wine64 wine ./osmcoverer_with_centers.exe -markers eligible_gyms.csv -grid=13
    ```

8. load output/output.geojson on geojson.io

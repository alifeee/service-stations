# service-stations

map of service stations in the uk from <https://motorwayservices.uk/> and Ireland from <https://motorwayservices.ie/>

view UK map on: <https://geojson.io/#data=data:text/x-url,https%3A%2F%2Fraw.githubusercontent.com%2Falifeee%2Fservice-stations%2Frefs%2Fheads%2Fmain%2Fservice-stations-uk.geojson>

view Ireland map on: (TODO)

also view on Google Maps / OSMAnd / Bing Maps / etc with...

## Import to different apps

### Google Maps

create a custom map ([how?](https://www.google.com/maps/about/mymaps/)) and import the `.kml` file above. Or, use this map I made: <https://www.google.com/maps/d/u/0/edit?mid=1llayKM-EKG17ENDg8x-3_Xv6-yNkTxU&usp=sharing>

### OSMAnd

Import as places to <https://osmand.net/>, by downloading the `.gpx` file and uploading it to places.

### Apple Maps

not sure yet...

## Generation

### UK Services

```bash
# get all possible pages from motorway services site
wget "https://motorwayservices.uk/elements/sitemap.xml" -O sitemap.xml
# format sitemap so there is one link per line
sudo apt install libxml2-utils 
xmllint --format sitemap.xml > sitemap_reformatted.xml
# search for links that do not look like "File:" or "History:" etc
cat sitemap_reformatted.xml | pcregrep -o1 "(https://motorwayservices.uk/[^:</]*)</loc>" | sort -h | uniq > pages.txt
# save all sites from sitemap
mkdir pages
while read site; do name=$(echo "${site}" | sed 's/\///g' | sed 's/https:motorwayservices.uk//g'); wget "${site}" -O "pages/${name}.html" --timeout=2 --tries=1; sleep 1; done <<< $(cat pages.txt)
# find postcodes from each HTML page, they look like:
#   </div><div class="infobyte2" style="border-style:solid;border-color: #00A34B"><b>Postcode:</b>
#   <p>LL57 4BG
pcregrep -ri "<p>[A-Z]{1,2}[0-9]{1,2} ?[0-9]{1,2}[A-Z]{1,2}" pages | pcregrep -o1 -o2 --om-separator="," "pages/(.*)\.html.*<p>([A-Z]{1,2}[0-9]{1,2} ?[0-9]{1,2}[A-Z]{1,2})" | sort -h > postcodes.csv 
# get lat/long coords from postcode using API from https://www.doogal.co.uk/BatchGeocoding
#   curl -s 'https://www.doogal.co.uk/GetPostcode/CV10%207DA' | jq '.latitude, .longitude' | paste -sd, -
while read line; do postcode=$(echo "${line}" | csvtool col 2 -); pcurlencoded=$(echo "${postcode}" | sed 's/ /%20/g'); curl -s "https://www.doogal.co.uk/GetPostcode/${pcurlencoded}" | jq '.latitude, .longitude' | paste -sd, - | echo "${line},"$(cat /dev/stdin) | tee -a coordinates.csv; done < postcodes.csv
# add URL to motorwayservices.uk
csvtool col 1 coordinates.csv | sed 's/^/http:\/\/motorwayservices.uk\//' | paste -d, coordinates.csv - > /tmp/coords
mv /tmp/coords coordinates.csv
# add header to CSV
sed -i '1s/^/name,postcode,latitude,longitude,URL\n/' coordinates.csv
# create geojson (see https://gist.github.com/alifeee/60e121a4b55ce1069b003e1d94f0e046)
git clone git@github.com:pvernier/csv2geojson.git
(cd csv2geojson/; go build main.go)
./csv2geojson/main coordinates.csv
# file now is -> coordinates.geojson !
```

### Ireland services

```bash
# Irish site has no sitemap, but lists all services when you search for them, as there are not many
wget "https://motorwayservices.ie/Services_Search?reply=yes&country=IE&road=Any&brands=Any&operator=Any&access=Any&rating=None&ratingt=All" -O services_search.html
# tidy HTML so that we can easily grep for links instead of parsing HTML
tidy services_search.html > temp.html; mv temp.html services_search.html
# find all links in table
cat services_search.html | pcregrep -o1 '<td.*<a href="(.*)"' | sort -h | uniq | sed 's/^/https:\/\/motorwayservices.ie/' > pages.txt
# save all sites locally
mkdir pages
while read site; do name=$(echo "${site}" | sed 's/\///g' | sed 's/https:motorwayservices.ie//g'); wget "${site}" -O "pages/${name}.html" --timeout=2 --tries=1; sleep 1; done <<< $(cat pages.txt)
# get eirecodes from each page
pcregrep -ri "<p>[A-Z0-9]{3} ?[A-Z0-9]{4}" pages | pcregrep -o1 -o2 --om-separator="," "pages/(.*)\.html.*<p>([A-Z0-9]{3} ?[A-Z0-9]{4})" | sort -h > eircodes.csv
# now, we cannot convert Eircodes to latlong coordinates easily, as Eircodes are proprietary(?)
#  so, what I did was manually paste each Eircode into Google Maps, refresh the page (to centre the page on the pin), and copy the coordinates from the URL
echo "do it manually :)"
# add URLs
csvtool col 1 eircodes.csv | sed 's/^/http:\/\/motorwayservices.ie\//' | paste -d, eircodes.csv - > /tmp/coords
mv /tmp/coords eircodes.csv
# add header to CSV
sed -i '1s/^/name,eircode,latitude,longitude,URL\n/' eircodes.csv
# create geojson (see https://gist.github.com/alifeee/60e121a4b55ce1069b003e1d94f0e046)
git clone git@github.com:pvernier/csv2geojson.git
(cd csv2geojson/; go build main.go)
./csv2geojson/main eircodes.csv
# file now is -> eircodes.geojson !
```

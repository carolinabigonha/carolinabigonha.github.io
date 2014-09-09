---
layout: post
title: "The basics of map visualization using d3.js"
date: 2014-08-18 10:58:53
introduction: For the past few months I have been playing with map visualizations using d3.js. After a while, I curated a few references and learned things worth sharing. So, in this (kind-of) tutorial, I'll cover how to find geographic files, load data and plot map visualizations for the Web.
---

## Finding map files
The first step into making map visualizations is obtaining the files with the desired geographic information. Nowadays, there are several sources for obtaining this kind of data, such as <a target="_blank" href="http://www.naturalearthdata.com/">Natural Earth Data</a>, a public domain map dataset, and <a target="_blank" href="http://geocommons.com/">GeoCommons</a>, a community for sharing open data and maps. 

Also, there is a growing effort of government institutions around the world to provide geographical information online. One may find U.S. maps at <a target="_blank" href="http://www.census.gov/">United States Census Bureau</a> and  <a target="_blank" href="http://www.data.gov/">data.gov</a> websites. For Brazilian maps, there are resources available at <a target="_blank" href="http://mapas.ibge.gov.br/">Instituto Brasileiro de Geografia e Estatística (IBGE)</a> website. Both U.S. Census Bureau and IBGE have FTP servers that may provide a more straightforward browsing experience -- <a target="blank" href="ftp://ftp2.census.gov/geo/tiger/TIGER2010/">U.S. Census Bureau's FTP server</a> and <a target="_blank" href="ftp://geoftp.ibge.gov.br/">IBGE's FTP server</a>.

Kindly, <a href="https://twitter.com/mbostock" target="_blank">@mbostock</a> provided us with tools for easily downloading maps from Natural Earth and U.S. Census Bureau -- available at his GitHub repository: <a target="_blank" href="https://github.com/mbostock/world-atlas">word-atlas</a> and <a href="https://github.com/mbostock/us-atlas" target="_blank">us-atlas</a>. Inspired by his effort, I've coded an easy way to download Brazilian maps from IBGE servers. It's also available on GitHub: <a href="https://github.com/carolinabigonha/br-atlas" target="_blank">br-atlas</a>.

##  Geographic data formats
Geographic files are available in different formats. In most cases, geographic repositories -- such as the ones mentioned above -- provide files in the <a href="http://www.esri.com/library/whitepapers/pdfs/shapefile.pdf" target="_blank">Shapefile format</a>. Specified by the <a href="http://www.esri.com/" target="_blank">Environmental Systems Research Institute (Esri)</a>, the shapefile format is a simple non-topological way of storing the geometric location and attribute information of geographic features. It represents its geographic features by points, lines and polygons and comprise in at least three files with the following extensions: `.shp`, `.shx` and `.dbf`. 

<a href="www.d3js.org" target="_blank">D3</a> supports, for cartographic visualizations, the <a target="_blank" href="http://geojson.org/">GeoJSON format</a>, which is a standard geographic data representation in JavaScript, based on JavaScript Object Notation (JSON). 

A GeoJSON always consists of a single object, with one required name/value member `type` and two optional members `crc` and `bbox`. The `type` value determines whether the GeoJSON represents a geometry, a feature or a collection of features. A <strong>geometry</strong> type is either one of: `Point` and `MultiPoint` (usually representing addresses and places), `LineString` and `MultiLineString` (usually representing streets, avenues and borders), `Polygon` and `MultiPolygon` (counties, states, pieces of land) and `GeometryCollection`, which is a collection of the mentioned geometries.  A <strong>feature</strong> type (`"type": "Feature"`) is a geometry object with additional properties and a <strong>collection of features</strong> (`"type": "FeatureCollection"`) is an array of features. 

The optional members `crc` and `bbox` determine, respectively: the Coordinate Reference System, with latitude and longitude by default; and the Bounding Box information, which is the coordinate range for geometries, features or feature collections of the GeoJSON object.

{% highlight javascript linenos %}
{ "type": "FeatureCollection",
  "features": [
    { "type": "Feature",
      "geometry": {"type": "Point", "coordinates": [102.0, 0.5]},
      "properties": {"prop0": "value0"}
    },
    { "type": "Feature",
      "geometry": {
        "type": "LineString",
        "coordinates": [
          [102.0, 0.0], [103.0, 1.0], [104.0, 0.0], [105.0, 1.0]
        ]
      },
      "properties": {
        "prop0": "value0",
        "prop1": 0.0
      }
    }
  ]
}
{% endhighlight %}
<span class="caption">Snippet 1. GeoJSON object with a collection of features that represent a `Point` and a `LineString`.</span>

Later on, Michael Bostock created the <a target="_blank" href="https://github.com/mbostock/topojson">TopoJSON</a> format, an extension of GeoJSON, that encodes topology. Instead of representing each geometry discretely, repeating shared line segments, like in GeoJSON, each geometry is a set of global <strong>arcs</strong>. For example, the coordinates of a shared border between two countries are represented only once in a TopoJSON file and twice in a GeoJSON file. By eliminating redundancy, typically, a TopoJSON file is 80% smaller then the GeoJSON equivalents. For the Web, it is an important aspect.

{% highlight javascript linenos %}
{
  "type": "Topology",
  "transform": {
    "scale": [0.036003600360036005, 0.017361589674592462],
    "translate": [-180, -89.99892578124998]
  },
  "objects": {
    "aruba": {
      "type": "Polygon",
      "arcs": [[0]],
      "id": 533
    }
  },
  "arcs": [
    [[3058, 5901], [0, -2], [-2, 1], [-1, 3], [-2, 3], [0, 3], 
    [1, 1], [1, -3], [2, -5], [1, -1]]
  ]
}
{% endhighlight %}
<span class="caption">Snippet 2. TopoJSON object containing a `Polygon` object named "aruba", with a parameter called `arcs` referencing the element of index 0 in the global `arcs` array.</span>

The `topology` type represents one of more geometries that share sequences of positions specified in `arcs`. All the GeoJSON geometry types are supported in TopoJSON. Moreover, the topology type may have a `transform` member, whose value must be a transform and a `bbox` member, whose value must be a bounding box array.

<strong>So, how should one choose between TopoJSON or GeoJSON?</strong> <i>"If you care about file size or topology, then use TopoJSON. If you don’t care about either, then use GeoJSON for simplicity’s sake."</i> said <a href="http://stackoverflow.com/questions/14740705/difference-between-geojson-and-topojson" target="_blank">Michael Bostock on a Stack Overflow thread</a>.

Basically, there are two benefits of using TopoJSON: file size and the use of topology -- <a href="http://bost.ocks.org/mike/simplify/" target="_blank">line simplification</a> and <a target="_blank" href="http://prag.ma/code/d3-cartogram/#birthrate/2011">form distortion</a>, for example. However, the TopoJSON format is more complex than GeoJSON: there's a need of converting it to GeoJSON in order to use `d3.geo.path` and its simplified representation of coordinates may introduce rounding errors. If it's the case of server-side manipulation with no real necessity of topological aspects, use a GeoJSON. Otherwise, use a TopoJSON.

## From Shapefile to GeoJSON/TopoJSON

For illustrating the whole process of finding the geographical file, converting it to GeoJSON/TopoJSON and plotting it using d3.js, I'll create a map visualization showing <i>São Paulo</i>'s counties and their population density.

For downloading the map files for <i>São Paulo</i> state and counties, I'll use <a href="https://github.com/carolinabigonha/br-atlas" target="_blank">br-atlas</a>. After cloning the repository and installing the dependencies (check out the instructions at the <a href="https://github.com/carolinabigonha/br-atlas/blob/master/README.md" target="_blank">repository page</a>), I'll run ```make topo/sp-counties.json``` inside `br-atlas/` directory. It'll make all the magic and generate two `sp-counties.json` files: a TopoJSON file inside of `topo/` and a GeoJSON file inside of `geo/`. 

Detailing it step by step, it first downloads the state map shapefiles available at <a target="_blank" href="ftp://geoftp.ibge.gov.br/">IBGE's FTP server</a> at the `/malhas_digitais/municipio_2010/` directory. The inflated the zip archive contains several files with the patterns `35MIE250GC_SIR.*`,  `35MEE250GC_SIR.*`,  `35UFE250GC_SIR.*` and  `35MUE250GC_SIR.*`. `MI` stands for microregion, `ME` for mesoregion, `UF` for state and `MU` for counties (that's what we're looking for). I store these files into the `tmp/sp/` directory.

We firstly convert the shapefiles into the GeoJSON format, using `ogr2ogr`, as follows: 

{% highlight bash %}
ogr2ogr -f GeoJSON geo/sp-counties.json tmp/sp/35MUE250GC_SIR.shp
{% endhighlight %}

Some of IBGE files are encoded with ISO-8859-1, so I convert all of them to UTF-8, as follows:

{% highlight bash %}
iconv -f ISO-8859-1 -t UTF-8 geo/sp-counties.json > geo/sp-counties.json.utf8
mv geo/sp-counties.json.utf8 geo/sp-counties.json
{% endhighlight %}

Finally, I convert it to TopoJSON, using the command line (check all possibilities <a target="_blank" href="https://github.com/mbostock/topojson/wiki/Command-Line-Reference">here</a>), as follows: 

{% highlight bash %}
node --max_old_space_size=8192 node_modules/.bin/topojson -q 1e6 --id-property=CD_GEOCODM -p name=NM_MUNICIP -o topo/sp-counties.json counties=geo/sp-counties.json
{% endhighlight %}

The parameter `--id-property=CD_GEOCODM` sets the `id` of the TopoJSON geometry as the geographical code `CD_GEOCODM`. The parameter `-p name=NM_MUNICIP` preserves only the name of the county as the `name` property of the geometry.

Comparing the generated GeoJSON and TopoJSON files, the first one has 31.9MB and the later only 4MB. Even though it is almost eight times smaller than GeoJSON, the TopoJSON file is still too large for loading client-side. So, I use the <a target="_blank" href="http://mapshaper.org/">MapShaper</a> tool for simplifying the lines. The resultant file has around 200KB and I store it as `sp-counties.min.json`.

## Let's make a map already!

The popularity density data that I'll use for plotting São Paulo's map is also available at <a href="http://cidades.ibge.gov.br/download/mapa_e_municipios.php?lang=&uf=sp" target="_blank">IBGE's website</a>. 

[TODO] -- carregar dados, plotar com d3.

## Further reading & Useful tools
* [Tutorial] For converting from Shapefile to GeoJSON, check <a target="_blank" href="http://vallandingham.me/shapefile_to_geojson.html">Jim Vallandingham's tutorial</a>. 
* [Tutorial] Michael Bostock on 
<a target="_blank" href="http://bost.ocks.org/mike/map/ ">Let's Make a Map</a>, 
<a target="_blank" href="http://bl.ocks.org/mbostock/4060606">Choropleth</a>, 
<a target="_blank" href="http://bost.ocks.org/mike/topology/ ">How to Infer Topology?</a> and  <a target="_blank" href="http://bl.ocks.org/mbostock/3711652">Projection Transitions</a>.
* [Tutorial] Interesting comparison of <a target="_blank" href="http://bl.ocks.org/syntagmatic/3711245">map projections</a>.
* [Tool] For simplifying Shapefile, GeoJSON or TopoJSON files, check the <a target="_blank" href="http://mapshaper.org/">MapShaper</a> tool. 
* [Book] Scott Murray's <a target="_blank" href="http://www.amazon.com/Interactive-Data-Visualization-Scott-Murray/dp/1449339735/">Interactive Data Visualizations for the Web</a>, Chapter 12 - Geomapping 
* [Book] Nathan Yau's <a href="http://www.amazon.com/Visualize-This-FlowingData-Visualization-Statistics/dp/0470944889" target="_blank">Visualize This</a>, Chapter 8 - Visualizing Spacial Relationships
* [Book] Alan M. MacEachren's <a href="http://www.amazon.com/How-Maps-Work-Representation-Visualization/dp/157230040X" target="_blank">How Maps Work: Representation, Visualization, and Design</a>
* [Spec] The <a target="_blank" href="https://github.com/mbostock/d3/wiki/Geo-Paths">Geo Paths specification</a> in D3 API Reference.






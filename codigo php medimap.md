medimapgye
==========
<?php 
function Distance($lat1, $lon1, $lat2, $lon2, $unit) { 
  
  $radius = 6378.137; // earth mean radius defined by WGS84
  $dlon = $lon1 - $lon2; 
  $distance = acos( sin(deg2rad($lat1)) * sin(deg2rad($lat2)) +  cos(deg2rad($lat1)) * cos(deg2rad($lat2)) * cos(deg2rad($dlon))) * $radius; 

  if ($unit == "K") {
  		return ($distance); 
  } else if ($unit == "M") {
    	return ($distance * 0.621371192);
  } else if ($unit == "N") {
    	return ($distance * 0.539956803);
  } else {
    	return 0;
  }
}

function array_sort($array, $on, $order=SORT_ASC)
{
    $new_array = array();
    $sortable_array = array();

    if (count($array) > 0) {
        foreach ($array as $k => $v) {
            if (is_array($v)) {
                foreach ($v as $k2 => $v2) {
                    if ($k2 == $on) {
                        $sortable_array[$k] = $v2;
                    }
                }
            } else {
                $sortable_array[$k] = $v;
            }
        }

        switch ($order) {
            case SORT_ASC:
                asort($sortable_array);
            break;
            case SORT_DESC:
                arsort($sortable_array);
            break;
        }

        foreach ($sortable_array as $k => $v) {
            $new_array[$k] = $array[$k];
        }
    }

    return $new_array;
}


$i=0;
if (isset($_GET['lat'])&&isset($_GET['lng'])){
	$miPos=array("lat"=>$_GET['lat'],"long"=>$_GET['lng']);
	}else{
	$miPos=array("lat"=>-2.132897,"long"=>-79.865910);
	}
$flag=0;
$id=0;
if (isset($_GET['IdEsp'])){$flag=1;
$id=$_GET['IdEsp'];};

$listHosp=array();
$CargaXML=simplexml_load_file('Hospitales.xml');

					  
					  for ($i=0;$i<($CargaXML->hospital->count());$i++){
						  $comp=$CargaXML->hospital[$i]->especialidad['id'];
						  if(($id== $comp)||($flag==0)){
	$listHosp[$i]['nombre']=$CargaXML->hospital[$i]->nombre;
	$listHosp[$i]['direccion']=$CargaXML->hospital[$i]->direccion;
	$listHosp[$i]['telefono']=$CargaXML->hospital[$i]->telefono;
	$listHosp[$i]['lat']=$CargaXML->hospital[$i]->lat;
	$listHosp[$i]['long']=$CargaXML->hospital[$i]->long;
	$listHosp[$i]['especialidad']=$CargaXML->hospital[$i]->especialidad;
	$listHosp[$i]['IdEspecialidad']=$CargaXML->hospital[$i]->especialidad['id'];
	$listHosp[$i]['Distancia']= Distance(doubleval($miPos['lat']), doubleval($miPos['long']),doubleval($listHosp[$i]['lat']), doubleval($listHosp[$i]['long']),"K");
	$listHosp[$i]['correo']=$CargaXML->hospital[$i]->correo;};
	}
					  
					  $listHosp=array_sort($listHosp, 'Distancia', SORT_ASC);
					  



?>
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<script
src="http://maps.googleapis.com/maps/api/js?key=AIzaSyDY0kkJiTPVd2U7aTOAwhc9ySH6oHxOIYM&sensor=true&language=es&region=ES">
</script>



<!--carga de mapa-->
<script>
var marcadores=new Array();
var InfoWin=new Array();
var ArrDistancias= new Array();
var Mipos = new google.maps.LatLng(<?php echo $miPos['lat'].','.$miPos['long'] ?>);
var distancia="";
var lastCercano;	


var rendererOptions = {
  draggable: true,
  suppressMarkers: true
};
var directionsDisplay = new google.maps.DirectionsRenderer(rendererOptions);;
var directionsService = new google.maps.DirectionsService();
var lastclick;
var lastInfowin;

function initialize()
{	 	
	var mapTypeIds = [];
            for(var type in google.maps.MapTypeId) {
                mapTypeIds.push(google.maps.MapTypeId[type]);
            }
            mapTypeIds.push("OSM");
			
var mapProp = {
  center: Mipos,
  zoom:10,
  mapTypeId:google.maps.MapTypeId.ROADMAP,
  mapTypeControlOptions: {
                    mapTypeIds: mapTypeIds
                }
  };
  
  
var map=new google.maps.Map(document.getElementById("googleMap"),mapProp);
//mapa y copyrights
map.mapTypes.set("OSM", new google.maps.ImageMapType({
                getTileUrl: function(coord, zoom) {
                    return "http://tile.openstreetmap.org/" + zoom + "/" + coord.x + "/" + coord.y + ".png";
                },
                tileSize: new google.maps.Size(256, 256),
                name: "OpenStreetMap",
                maxZoom: 18
            }));
			
copyrightDiv = document.createElement("div")
copyrightDiv.id = "map-copyright"
copyrightDiv.style.fontSize = "11px"
copyrightDiv.style.fontFamily = "Arial, sans-serif"
copyrightDiv.style.margin = "0 2px 2px 0"
copyrightDiv.style.whiteSpace = "nowrap"
map.controls[google.maps.ControlPosition.BOTTOM_RIGHT].push(copyrightDiv)

copyrights = {}
copyrights["OSM"] = "Map data&copy <a target=\"_blank\" href=\"http://www.openstreetmap.org/\">OpenStreetMap</a> contributors, Imagery&copy <a target=\"_blank\" href=\"http://www.openstreetmap.org/\">OpenStreetMap</a>"

function onMapTypeIdChanged()
{
    newMapType = map.getMapTypeId()

    copyrightDiv = document.getElementById("map-copyright")
    if(newMapType in copyrights)
        copyrightDiv.innerHTML = copyrights[newMapType]
    else
        copyrightDiv.innerHTML = ""
}

google.maps.event.addListener(map, "maptypeid_changed", onMapTypeIdChanged)
directionsDisplay.setMap(map);
  directionsDisplay.setPanel(document.getElementById('directionsPanel'));
// Call once so that the correct copyright notice is set for
// the initially selected map type
setTimeout(onMapTypeIdChanged, 50)
//Marcador de posición inicial
var markerini = new google.maps.Marker({

                        position: Mipos
                                , map: map  
								, title:"Mi posición"                              
				, icon: "images/mkverde.png"
				,draggable:true
                      });
				

//PHP marcadores de hospitales

<?php
$listHosp;
$i=1;
foreach($listHosp as $hospital){
	if($i==1){
	echo 'var hosp'.$i.' = new google.maps.Marker({

                        position: new google.maps.LatLng('.$hospital['lat'].','.$hospital['long'].')
                                , map: map  
								, title:"Hospital más cercano"                              
				, icon: "images/mkrojo.png"
                      });
					  lastCercano=hosp'.$i.';
					  lastclick=hosp'.$i.';
					  ';
					  }else{						  
					echo 'var hosp'.$i.' = new google.maps.Marker({

                        position: new google.maps.LatLng('.$hospital['lat'].','.$hospital['long'].')
                                , map: map  
								, title:"Hospital cercano"                              
				, icon: "images/mkazul.png"
                      });
					  ';	  
						  };
			echo "
			var contentString".$i."a =".'"'."<meta http-equiv='Content-Type' content='text/html;charset=UTF-8'/>".'"'."+
".'"'."<div style='font-size: 8pt; font-family: verdana; width: 300px; height:100px;'>".'"'."+
".'"'."<table border='1' cellpadding='0' cellspacing='0' style='width: 300px; height:100px;'>".'"'."+
".'"'."<tr><td colspan='2' align='center'><b>".$hospital['nombre']."</b></td></tr>".'"'."+
".'"'."<tr><td colspan='2'>".$hospital['direccion']."</td></tr>".'"'."+
".'"'."<tr><td colspan ='2'>".$hospital['telefono']."</td></tr>".'"'."+
".'"'."<tr><td colspan='2'>Especialidad: ".$hospital['especialidad']."</td></tr>".'"'."+
".'"'."<tr>".'"'."+
".'"'."<td width='116'>Distancia: ".'"'.";
var contentString".$i."b =".'"'."</td></tr>".'"'."+
".'"'."</table>".'"'."+
".'"'."</div>".'";
';

echo "var popup".$i." = new google.maps.InfoWindow({

                                content: contentString".$i."a

                      });
google.maps.event.addListener(hosp".$i.", 'click', function(){
	CloseInfowindows();
 
  lastclick =  hosp".$i.";
  lastInfowin=popup".$i.";
 
   var request = {
    origin:  markerini.getPosition(),
    destination: new google.maps.LatLng(".$hospital['lat'].','.$hospital['long']."),
	region: 'es',

    travelMode: google.maps.DirectionsTravelMode.DRIVING
  };
   directionsService.route(request, function(response, status) {
    if (status == google.maps.DirectionsStatus.OK) {
      directionsDisplay.setDirections(response);
    }
	computeTotalDistance(directionsDisplay.directions);
	 popup".$i.".setContent(contentString".$i."a + distancia +contentString".$i."b);
  });
   popup".$i.".open(map, hosp".$i.");
  });
  marcadores[".($i-1)."]=hosp".$i.";
  InfoWin[".($i-1)."]=popup".$i.";
  ";			  
	
	++$i;
	$la=$hospital['lat'];
	$lo=$hospital['long'];
	}

?>



function computeTotalDistance(result) {
  var total = 0;
  var myroute = result.routes[0];
  for (var i = 0; i < myroute.legs.length; i++) {
    total += myroute.legs[i].distance.value;
  }
  total = total / 1000.
  document.getElementById('total').innerHTML = total + ' km';
  distancia=total + ' km';
}
var inibounds= <?php echo 'new google.maps.LatLng('.$la.','.$lo.')' ?>;
map.setCenter(Mipos);
//cerrar todas las infowindows
function CloseInfowindows() {
  for (var mkey in InfoWin) {
    var mobj = InfoWin[mkey];
    mobj.close();
  }
};


// Try HTML5 geolocation
	function onSuccess(position){
   Mipos = new google.maps.LatLng(position.coords.latitude, position.coords.longitude);
   markerini.setPosition(Mipos);
   map.panTo(Mipos);
   var request = {
    origin:  markerini.getPosition(),
    destination: lastclick.getPosition(),
	region: 'es',

    travelMode: google.maps.DirectionsTravelMode.DRIVING
  };
  lastInfowin.close();
  
  
   directionsService.route(request, function(response, status) {
    if (status == google.maps.DirectionsStatus.OK) {
      directionsDisplay.setDirections(response);
    }
	computeTotalDistance(directionsDisplay.directions);
  }); 
  marcadores[localkey].setIcon("images/mkrojo.png");
		marcadores[localkey].setTitle("Hospital más cercano");
		lastCercano.setIcon("images/mkazul.png");
		lastCercano.setTitle("Hospital cercano");
		lastCercano=marcadores[localkey];
   
  };
  function onError(error) {
    alert('code: ' + error.code + '\n' +
        'message: ' + error.message + '\n');
};
function getLocation()
  { 
  if (navigator.geolocation)
    {
    navigator.geolocation.getCurrentPosition(onSuccess, onError);
	    };
  };
  getLocation();
  window.setInterval(function(){getLocation();},30000);
var bounds = new google.maps.LatLngBounds;
bounds.extend(inibounds);
bounds.extend(Mipos);
map.fitBounds(bounds);
google.maps.event.addListener(markerini, "position_changed", function() {
	/*bounds = new google.maps.LatLngBounds;
bounds.extend(inibounds);
bounds.extend(Mipos);
map.fitBounds(bounds);*/



var localkey=MinArrDistPos();
		marcadores[localkey].setIcon("images/mkrojo.png");
		marcadores[localkey].setTitle("Hospital más cercano");
		lastCercano.setIcon("images/mkazul.png");
		lastCercano.setTitle("Hospital cercano");
		lastCercano=marcadores[localkey];
				
	});
	//ini
	function MinArrDistMarker()
  {
	 
	   for (var mkey in marcadores) {
    var mobj = marcadores[mkey];
	ArrDistancias[mkey]=Dist(mobj.getPosition().lat(), mobj.getPosition().lng(), markerini.getPosition().lat(), markerini.getPosition().lng());
	
  }
	  Array.min = function( array ){
    return Math.min.apply( Math, array );
		};

		var value = Math.min.apply(Math, ArrDistancias);
		var key = ArrDistancias.indexOf(Math.min.apply(Math, ArrDistancias));
		for (var mkey in ArrDistancias) {
			if(ArrDistancias[mkey]==value){key = mkey};
			};
			return key;
	  };
	  //mipos
	function MinArrDistPos()
  {
	 
	   for (var mkey in marcadores) {
    var mobj = marcadores[mkey];
  	ArrDistancias[mkey]=Dist(mobj.getPosition().lat(), mobj.getPosition().lng(), Mipos.lat(), Mipos.lng());
  }
	  Array.min = function( array ){
    return Math.min.apply( Math, array );
		};

		var value = Math.min.apply(Math, ArrDistancias);
		var key = ArrDistancias.indexOf(Math.min.apply(Math, ArrDistancias));
		for (var mkey in ArrDistancias) {
			if(ArrDistancias[mkey]==value){key = mkey};
			}
		return key;
	  };
	  
	  
	  
	  
	
	function Dist(lat1, lon1, lat2, lon2)
  {
  rad = function(x) {return x*Math.PI/180;}

  var R     = 6378.137;                          //Radio de la tierra en km
  var dLat  = rad( lat2 - lat1 );
  var dLong = rad( lon2 - lon1 );

  var a = Math.sin(dLat/2) * Math.sin(dLat/2) + Math.cos(rad(lat1)) * Math.cos(rad(lat2)) * Math.sin(dLong/2) * Math.sin(dLong/2);
  var c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
  var d = R * c;

  return d.toFixed(3);                      //Retorna tres decimales
}
google.maps.event.addListener(markerini, "dragend", function() {
		var request = {
    origin:  markerini.getPosition(),
    destination: lastclick.getPosition(),
	region: 'es',

    travelMode: google.maps.DirectionsTravelMode.DRIVING
  };
  lastInfowin.close();
   directionsService.route(request, function(response, status) {
    if (status == google.maps.DirectionsStatus.OK) {
      directionsDisplay.setDirections(response);
    }
	computeTotalDistance(directionsDisplay.directions);
  });

		var localkey=MinArrDistMarker();
		marcadores[localkey].setIcon("images/mkrojo.png");
		marcadores[localkey].setTitle("Hospital más cercano");
		lastCercano.setIcon("images/mkazul.png");
		lastCercano.setTitle("Hospital cercano");
		lastCercano=marcadores[localkey];

		});	
	
}
google.maps.event.addDomListener(window, 'load', initialize);
window.setInterval("getLocation",30000);

</script>
<title>MediMap</title>
 <style>
      html, body, #googleMap {
        height: 95%;
        margin: 0px;
        padding: 0px
      }
    </style>
</head>
<body>
<div style="background-image:url(images/medimap%20banner.jpg); height:112px; width:100%"></div>
<div id="googleMap" style="float:left;width: 70%; height: 95%"></div>
<div id="directionsPanel" style="float:right;width:30%;height: 95%">
    <p>Distancia Total: <span id="total"></span></p>
    </div>
</body>
</html>

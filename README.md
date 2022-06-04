- 👋 Hi, I’m @Parpaya
- Here some code in GEE. Enjoy.

var incendio = 
    /* color: #0b4a8b */
    /* shown: false */
    /* displayProperties: [
      {
        "type": "rectangle"
      }
    ] */
    ee.Geometry.Polygon(
        [[[-5.354390958232031, 36.643781631981426],
          [-5.354390958232031, 36.37113787979427],
          [-4.923177579325781, 36.37113787979427],
          [-4.923177579325781, 36.643781631981426]]], null, false),
    s2Colection = ee.ImageCollection("COPERNICUS/S2_SR_HARMONIZED"),
    RGBVisParam = {"opacity":1,"bands":["B4","B3","B2"],"max":3000,"gamma":1.524};



// Vamos a filtrar la colección de Sentinel 2 para las fechas del incendio
// y que sólo seleccione las imágenes de la zona de interés. 
var s2Filtro = s2Colection.filterDate('2021-09-15', '2021-09-30')
                          .filterBounds(incendio)
                          .filter(ee.Filter.lte('CLOUDY_PIXEL_PERCENTAGE', 5));


// Filtro de nubes basado en la banda SCL de S2_SR
// Elimina las capas de nubes, sombras, hielo, agua y píxeles saturados 
var filtroNubes = function(s2Image) {
  var scl = s2Image.select('SCL');
  var mask = scl.lt(6).and(scl.neq(3)).and(scl.neq(1));
  return s2Image.updateMask(mask);
};

// Aplicamos el filtro de nubes y seleccionamos las bandas de interés
// Podemos seleccionar distintas bandas y comparar el resultado
// Seleccioanr demasiadas bandas nos puede dar error de memoria por lo que jugamos con el número de bandas y número de pixels de entrenamiento
var input = s2Filtro.map(filtroNubes).median().select(['B2', 'B3', 'B4' , 'B5', 'B7', 'B8' , 'B11', 'B12']);

// Añadimos al mapa las imágenes de NDVI y visible
Map.addLayer(input, RGBVisParam, 'RGB'); 


// Centramos el mapa en el sector del incendio
Map.centerObject(incendio, 11);

// Generamos un panel que ocupe una quinta parte de la interfaz
var panel = ui.Panel({style: {width:'20%'}});
// Y lo añadimos a la interfaz general
ui.root.insert(0, panel);

// Definimos un título 
var intro = ui.Label('Pulsa en el mapa para ver las coordenadas',
  {fontWeight: 'bold', fontSize: '24px', margin: '10px 5px'}
);
// Y un subtítulo o descripción
var subtitle = ui.Label('Las coordenadas son:', {});

// Añadimos los elmentos al panel, incluyendo el botón 
panel.add(intro).add(subtitle);

// Añadimos un pequeño panel para las coordenadas
var lon = ui.Label();
var lat = ui.Label();
panel.add(ui.Panel([lon, lat], ui.Panel.Layout.flow('horizontal')));

// Mostramos un punto y sus coordenadas cuando pulsamos en el mapa.
// La función toma como parametro las coordenadas 
Map.onClick(function(coords) {
  // Actulizamos las etiquetas para que se muestren con estos textos
  lat.setValue('lat: ' + coords.lat.toFixed(3));
  lon.setValue('lon: ' + coords.lon.toFixed(3));
  // Creamos una geometría de punto y la añadimos al mapa
  // Esta geometría es la que nos sirve para generar las coordenadas
  var point = ee.Geometry.Point(coords.lon, coords.lat);
  var dot = ui.Map.Layer(point, {color: 'FF0000'}, 'Punto');
  Map.layers().set(1, dot);
  var inspector = ui.Panel([ui.Label(coords.lon,{fontSize: '10px', margin: '1px 1px'})]);
  inspector.add(ui.Label(coords.lat,{fontSize: '10px', margin: '1px 1px'}));
  Map.add(inspector);
});

// Definimos que el cursor se muestre como una cruz para saber que esperamos un input del usuario
Map.style().set('cursor', 'crosshair');


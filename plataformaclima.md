// Carregamento das fectures 
var cidades = ee.FeatureCollection('projects/ee-joeltoconpp/assets/Planalto_cidades');

// Menu dos nomes dos municípios 
var nomesCidades = ['Belo Campo', 'Boa Nova', 'Caatiba', 'Cândido Sales', 'Dário Meira', 'Manoel Vitorino', 'Nova Canaã', 'Planalto', 'Poções', 'Anagé', 'Barra do Choça', 'Vitória da Conquista'];

// Função para calcular NDVI
function calcularNDVI(image) {
    var ndvi = image.normalizedDifference(['SR_B5', 'SR_B4']).rename('NDVI');
    return image.addBands(ndvi);
}

// Função para calcular NDWI
function calcularNDWI(image) {
    var ndwi = image.normalizedDifference(['SR_B3', 'SR_B5']).rename('NDWI');
    return image.addBands(ndwi);
}

// Visualização
var ndviParams = {min: -1, max: 1, palette: ['red', 'yellow', 'green']};
var ndwiParams = {min: -1, max: 1, palette: ['darkblue', 'blue', 'lightblue', 'yellow', 'orange', 'red']};
var precipParams = {min: 0, max: 100, palette: ['blue', 'green', 'yellow', 'red']};
var tempParams = {min: 0, max: 40, palette: ['darkred', 'red', 'lightcoral']}; // Ajuste na paleta de cores para temperatura

// Painel de controle
var painel = ui.Panel({
  style: {width: '300px', position: 'top-left', padding: '10px'}
});
// Título
painel.add(ui.Label('Análises climáticas dos municípios do planalto da Conquista', {fontWeight: 'bold', fontSize: '18px', margin: '10px 0'}));

// Menu de cidades 
var seletorCidades = ui.Select({
  items: nomesCidades,
  placeholder: 'Selecione uma cidade',
  onChange: function(nomeCidade) {
    var cidadeSelecionada = cidades.filter(ee.Filter.eq('NM_MUN', nomeCidade));
    Map.centerObject(cidadeSelecionada, 10);
    atualizarMapa();
  }
});
painel.add(ui.Label('Selecione uma cidade:'));
painel.add(seletorCidades);

// Escolher as datas 
var inicioInput = ui.Textbox({
  placeholder: 'AAAA-MM-DD',
  value: '2020-01-01',
  onChange: atualizarMapa
});
var fimInput = ui.Textbox({
  placeholder: 'AAAA-MM-DD',
  value: '2021-01-01',
  onChange: atualizarMapa
});
painel.add(ui.Label('Data de início:'));
painel.add(inicioInput);
painel.add(ui.Label('Data de fim:'));
painel.add(fimInput);

// Menu de índices 
var indices = ['NDVI', 'NDWI', 'Precipitação', 'Temperatura'];
var seletorIndices = ui.Select({
  items: indices,
  placeholder: 'Selecione um índice',
  onChange: atualizarMapa
});
painel.add(ui.Label('Selecione um índice:'));
painel.add(seletorIndices);

// Botão de gráficos 
var gerarGraficosButton = ui.Button({
  label: 'Gerar Gráficos',
  onClick: gerarGraficos
});
painel.add(gerarGraficosButton);

// Atualizar mapa 
function atualizarMapa() {
  var inicio = inicioInput.getValue();
  var fim = fimInput.getValue();
  var nomeCidade = seletorCidades.getValue();
  var indice = seletorIndices.getValue();
  if (!nomeCidade || !indice) return;
  
  var cidadeSelecionada = cidades.filter(ee.Filter.eq('NM_MUN', nomeCidade));
  var geometriaCidade = cidadeSelecionada.geometry();
  
  // Limpar camadas anteriores 
  Map.layers().reset();
  
  if (indice === 'NDVI') {
    var ndvi = ee.ImageCollection('LANDSAT/LC08/C02/T1_L2')
      .filterBounds(geometriaCidade)
      .filterDate(inicio, fim)
      .map(calcularNDVI)
      .mean()
      .clip(geometriaCidade);
    Map.addLayer(ndvi.select('NDVI'), ndviParams, 'NDVI');
  } else if (indice === 'NDWI') {
    var ndwi = ee.ImageCollection('LANDSAT/LC08/C02/T1_L2')
      .filterBounds(geometriaCidade)
      .filterDate(inicio, fim)
      .map(calcularNDWI)
      .mean()
      .clip(geometriaCidade);
    Map.addLayer(ndwi.select('NDWI'), ndwiParams, 'NDWI');
  } else if (indice === 'Precipitação') {
    var precip = ee.ImageCollection('UCSB-CHG/CHIRPS/DAILY')
      .filterBounds(geometriaCidade)
      .filterDate(inicio, fim)
      .select('precipitation')
      .mean()
      .clip(geometriaCidade);
    Map.addLayer(precip, precipParams, 'Precipitação');
  } else if (indice === 'Temperatura') {
    var temperatura = ee.ImageCollection('ECMWF/ERA5/DAILY')
      .filterBounds(geometriaCidade)
      .filterDate(inicio, fim)
      .select('mean_2m_air_temperature')
      .map(function(image) {
        return image.subtract(273.15).rename('temperature_C'); // Converter de Kelvin para Celsius
      })
      .map(function(image) {
        return image.set('system:time_start', image.get('system:time_start'));
      })
      .mean()
      .clip(geometriaCidade);
    Map.addLayer(temperatura, tempParams, 'Temperatura');
  }
}

// Geração de gráficos 
function gerarGraficos() {
  var inicio = inicioInput.getValue();
  var fim = fimInput.getValue();
  var nomeCidade = seletorCidades.getValue();
  
  if (!nomeCidade) return;
  
  var cidadeSelecionada = cidades.filter(ee.Filter.eq('NM_MUN', nomeCidade));
  var geometriaCidade = cidadeSelecionada.geometry();
  
  var precipCollection = ee.ImageCollection('UCSB-CHG/CHIRPS/DAILY')
    .filterBounds(geometriaCidade)
    .filterDate(inicio, fim)
    .select('precipitation');
  
  var tempCollection = ee.ImageCollection('ECMWF/ERA5/DAILY')
    .filterBounds(geometriaCidade)
    .filterDate(inicio, fim)
    .select('mean_2m_air_temperature')
    .map(function(image) {
      return image.subtract(273.15).rename('temperature_C').set('system:time_start', image.get('system:time_start')); // Converter de Kelvin para Celsius e adicionar system:time_start
    });
  
  var evapotranspiracaoCollection = ee.ImageCollection('MODIS/006/MOD16A2')
    .filterBounds(geometriaCidade)
    .filterDate(inicio, fim)
    .select('ET');
  
  // Precipitação
  var precipChart = ui.Chart.image.series({
    imageCollection: precipCollection,
    region: geometriaCidade,
    reducer: ee.Reducer.mean(),
    scale: 5000
  }).setOptions({
    title: 'Precipitação ao longo do tempo',
    vAxis: {title: 'Precipitação (mm)'},
    lineWidth: 1,
    pointSize: 3
  });
  
  // Temperatura
  var tempChart = ui.Chart.image.series({
    imageCollection: tempCollection,
    region: geometriaCidade,
    reducer: ee.Reducer.mean(),
    scale: 5000
  }).setOptions({
    title: 'Temperatura ao longo do tempo',
    vAxis: {title: 'Temperatura (°C)'},
    lineWidth: 1,
    pointSize: 3
  });
  
  // Evapotranspiração
  var evapotranspiracaoChart = ui.Chart.image.series({
    imageCollection: evapotranspiracaoCollection,
    region: geometriaCidade,
    reducer: ee.Reducer.mean(),
    scale: 5000
  }).setOptions({
    title: 'Evapotranspiração ao longo do tempo',
    vAxis: {title: 'Evapotranspiração (mm)'},
    lineWidth: 1,
    pointSize: 3
  });
  
  // Adicionar gráficos ao painel
  painelGraficos.clear();
  painelGraficos.add(precipChart);
  painelGraficos.add(tempChart);
  painelGraficos.add(evapotranspiracaoChart);
}

// Adicionar painel de gráficos ao painel de controle
var painelGraficos = ui.Panel();
painel.add(painelGraficos);

// Adicionar painel de controle ao mapa 
ui.root.insert(0, painel);

// Clique no mapa 
Map.onClick(function(coords) {
  var nomeCidade = seletorCidades.getValue();
  if (!nomeCidade) return;
  
  var cidadeSelecionada = cidades.filter(ee.Filter.eq('NM_MUN', nomeCidade));
  var geometriaCidade = cidadeSelecionada.geometry();
  
  var ponto = ee.Geometry.Point([coords.lon, coords.lat]);
  
  var temperaturaCollection = ee.ImageCollection('ECMWF/ERA5/DAILY')
    .filterBounds(geometriaCidade)
    .filterDate(inicioInput.getValue(), fimInput.getValue())
    .select('mean_2m_air_temperature')
    .map(function(image) {
      return image.subtract(273.15).rename('temperature_C'); // Converter de Kelvin para Celsius
    });
  
  var precipCollection = ee.ImageCollection('UCSB-CHG/CHIRPS/DAILY')
    .filterBounds(geometriaCidade)
    .filterDate(inicioInput.getValue(), fimInput.getValue())
    .select('precipitation');
  
  var temperaturaPonto = temperaturaCollection.mean().reduceRegion({
    reducer: ee.Reducer.mean(),
    geometry: ponto,
    scale: 30
  }).get('temperature_C');
  
  var precipPonto = precipCollection.mean().reduceRegion({
    reducer: ee.Reducer.mean(),
    geometry: ponto,
    scale: 30
  }).get('precipitation');
  
  temperaturaPonto.evaluate(function(temperatura) {
    precipPonto.evaluate(function(precip) {
      var info = 'Temperatura: ' + temperatura.toFixed(2) + ' °C\nPrecipitação: ' + precip.toFixed(2) + ' mm';
      Map.add(ui.Label(info, {position: 'bottom-right', fontWeight: 'bold'}));
    });
  });
});

# 插件使用说明
## volkovlabs-echarts-panel
```
make build_go
```
```
./bin/darwin-amd64/grafana-server web
yarn start
```

http://localhost:3000

https://grafana.com/grafana/plugins/volkovlabs-echarts-panel/?tab=installation


https://github.com/grafana/business-charts

```
cd plugins-bundled/internal/input-datasource
yarn install
yarn build
```

```sql
-- 创建风向数据表
CREATE TABLE wind_data (
  id INT AUTO_INCREMENT PRIMARY KEY,
  ts DATETIME NOT NULL,         -- 时间戳
  wind_speed FLOAT NOT NULL,    -- 风速（节）
  wind_dir FLOAT NOT NULL       -- 风向（角度，单位度或弧度）
);

-- 插入示例数据
INSERT INTO wind_data (ts, wind_speed, wind_dir) VALUES
('2025-11-14 08:00:00', 6.19, 54.7),
('2025-11-14 09:00:00', 8.20, 120.0),
('2025-11-14 10:00:00', 11.19, 180.0),
('2025-11-14 11:00:00', 17.19, 270.0),
('2025-11-14 12:00:00', 3.19, 350.0);
```

## 基础调试
```sql
SELECT ts AS "time", wind_speed, wind_dir FROM wind_data ORDER BY ts
```
```js
console.log('windData:', data);

return {
  baseOption: {
    // ...你的 option 配置...
  }
};
```
**console.log是在F12控制台查看输出结果**

## 折线图

```sql
SELECT ts AS "time", wind_speed, wind_dir FROM wind_data ORDER BY ts
```
```js
const series = data.series.map(s => {
  const sData = s.fields.find(f => f.type === 'number').values.buffer;
  const sTime = s.fields.find(f => f.type === 'time').values.buffer;

  return {
    name: s.name,
    type: 'line',
    showSymbol: false,
    areaStyle: { opacity: 0.1 },
    lineStyle: { width: 1 },
    data: sData.map((d, i) => [sTime[i], d]
    ),
  };
});

const axisOption = {
  axisTick: { show: false },
  axisLine: { show: false },
  axisLabel: { color: 'rgba(128, 128, 128, .9)' },
  splitLine: { lineStyle: { color: 'rgba(128, 128, 128, .2)' } },
};

return {
  backgroundColor: 'transparent',
  tooltip: { trigger: 'axis' },
  legend: {
    left: '0',
    bottom: '0',
    data: data.series.map(s => s.name),
    textStyle: { color: 'rgba(128, 128, 128, .9)' },
  },
  xAxis: Object.assign({ type: 'time' }, axisOption),
  yAxis: Object.assign({ type: 'value', min: 'dataMin' }, axisOption),
  grid: { left: 0, right: 16, top: 6, bottom: 24, containLabel: true },
  series,
};
```

## 风向图
```sql
SELECT ts AS "time", wind_speed, wind_dir FROM wind_data ORDER BY ts
```

```js
const s = data.series[0];
const timeArr = s.fields.find(f => f.name === 'Time').values.buffer;
const speedArr = s.fields.find(f => f.name === 'wind_speed').values.buffer;
const dirArr = s.fields.find(f => f.name === 'wind_dir').values.buffer;

// 风向图标 series
const windDirectionSeries = {
  name: '风向',
  type: 'custom',
  renderItem: function (param, api) {
    const point = api.coord([api.value(0), api.value(1)]);
    return {
      type: 'path',
      shape: {
        pathData: 'M31 16l-15-15v9h-26v12h26v9z',
        x: -12 / 2,
        y: -12 / 2,
        width: 12,
        height: 12,
      },
      rotation: api.value(2) * Math.PI / 180,
      position: point,
      style: api.style({
        stroke: '#555',
        lineWidth: 1,
      }),
    };
  },
  encode: { x: 0, y: 1 },
  data: timeArr.map((t, i) => [t, speedArr[i], dirArr[i]]),
  z: 10,
};

// 风速折线 series
const windSpeedLineSeries = {
  name: '风速',
  type: 'line',
  data: timeArr.map((t, i) => [t, speedArr[i]]),
  showSymbol: false,
  areaStyle: { opacity: 0.1 },
  lineStyle: { width: 1 },
  z: 5,
};

const axisOption = {
  axisTick: { show: false },
  axisLine: { show: false },
  axisLabel: { color: 'rgba(128, 128, 128, .9)' },
  splitLine: { lineStyle: { color: 'rgba(128, 128, 128, .2)' } },
};

return {
  backgroundColor: 'transparent',
  tooltip: {
    trigger: 'axis',
    formatter: function (params) {
      return [
        echarts.format.formatTime('yyyy-MM-dd', params[0].value[0]) +
        ' ' +
        echarts.format.formatTime('hh:mm', params[0].value[0]),
        '风速：' + params[0].value[1],
        '风向：' + (params[0].value[2] !== undefined ? params[0].value[2] + '°' : ''),
      ].join('<br>');
    },
  },
  legend: {
    left: '0',
    bottom: '0',
    data: ['风向', '风速'],
    textStyle: { color: 'rgba(128, 128, 128, .9)' },
  },
  xAxis: Object.assign({ type: 'time' }, axisOption),
  yAxis: Object.assign({ type: 'value', min: 'dataMin' }, axisOption),
  grid: { left: 0, right: 16, top: 6, bottom: 24, containLabel: true },
  series: [windDirectionSeries, windSpeedLineSeries],
};
```

```js
const s = data.series[0];
const timeArr = s.fields.find(f => f.name === 'Time').values.buffer;
const speedArr = s.fields.find(f => f.name === 'wind_speed').values.buffer;
const dirArr = s.fields.find(f => f.name === 'wind_dir').values.buffer;

// 颜色点
const colorStops = [
  { val: 0, color: [105, 182, 126] },   // #69b67e
  { val: 0.5, color: [253, 206, 53] },  // #fdce35
  { val: 1, color: [253, 104, 93] }     // #fd685d
];

// 计算风速归一化
const minSpeed = Math.min(...speedArr);
const maxSpeed = Math.max(...speedArr);
function normalize(val) {
  if (maxSpeed === minSpeed) return 0;
  return (val - minSpeed) / (maxSpeed - minSpeed);
}

// 手动插值颜色
function getGradientColor(val) {
  const t = normalize(val);
  let left = colorStops[0], right = colorStops[colorStops.length - 1];
  for (let i = 1; i < colorStops.length; i++) {
    if (t <= colorStops[i].val) {
      left = colorStops[i - 1];
      right = colorStops[i];
      break;
    }
  }
  const ratio = (t - left.val) / (right.val - left.val);
  const rgb = left.color.map((c, i) => Math.round(c + (right.color[i] - c) * ratio));
  return `rgb(${rgb[0]},${rgb[1]},${rgb[2]})`;
}
const arrowSize = 36;
// 风向图标 series
const windDirectionSeries = {
  name: '风向',
  type: 'custom',
  renderItem: function (param, api) {
    const point = api.coord([api.value(0), api.value(1)]);
    const speed = api.value(1);
    const color = getGradientColor(speed);
    return {
      type: 'path',
      shape: {
        pathData: 'M31 16l-15-15v9h-26v12h26v9z',
        x: -arrowSize / 2,
        y: -arrowSize / 2,
        width: arrowSize,
        height: arrowSize,
      },
      rotation: api.value(2) * Math.PI / 180,
      position: point,
      style: api.style({
        stroke: color,
        fill: color,
        lineWidth: 1,
      }),
    };
  },
  encode: { x: 0, y: 1 },
  data: timeArr.map((t, i) => [t, speedArr[i], dirArr[i]]),
  z: 10,
};

// 折线渐变色
const lineGradient = {
  type: 'linear',
  x: 0, y: 0, x2: 1, y2: 0,
  colorStops: [
    { offset: 0, color: '#69b67e' },
    { offset: 0.5, color: '#fdce35' },
    { offset: 1, color: '#fd685d' }
  ]
};

// 风速折线 series
const windSpeedLineSeries = {
  name: '风速',
  type: 'line',
  data: timeArr.map((t, i) => [t, speedArr[i]]),
  showSymbol: false,
  areaStyle: { opacity: 0.1 },
  lineStyle: { width: 2, color: lineGradient },
  z: 5,
};

const axisOption = {
  axisTick: { show: false },
  axisLine: { show: false },
  axisLabel: { color: 'rgba(128, 128, 128, .9)' },
  splitLine: { lineStyle: { color: 'rgba(128, 128, 128, .2)' } },
};

return {
  backgroundColor: 'transparent',
  tooltip: {
    trigger: 'axis',
    formatter: function (params) {
      return [
        echarts.format.formatTime('yyyy-MM-dd', params[0].value[0]) +
        ' ' +
        echarts.format.formatTime('hh:mm', params[0].value[0]),
        '风速：' + params[0].value[1],
        '风向：' + (params[0].value[2] !== undefined ? params[0].value[2] + '°' : ''),
      ].join('<br>');
    },
  },
  legend: {
    left: '0',
    bottom: '0',
    data: ['风向', '风速'],
    textStyle: { color: 'rgba(128, 128, 128, .9)' },
    // 渐变条图例（低版本不支持自定义icon渐变，建议用文字标注 low/high）
    formatter: function (name) {
      if (name === '风速') {
        return 'low  风速  high';
      }
      return name;
    }
  },
  xAxis: Object.assign({ type: 'time' }, axisOption),
  yAxis: Object.assign({ type: 'value', min: 'dataMin' }, axisOption),
  grid: { left: 0, right: 16, top: 6, bottom: 24, containLabel: true },
  series: [windDirectionSeries, windSpeedLineSeries],
};
```
## 本地上安装PluginsGoogle BigQuery
```
grafana-cli plugins install simpod-json-datasource
```

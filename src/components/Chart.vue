<template>
  <div :id="chartId"></div>
</template>

<script>
import {
  AxisScrollStrategies,
  AxisTickStrategies,
  AutoCursorModes,
  ColorHEX,
  emptyFill,
  SolidFill,
  emptyLine,
  SolidLine,
  lightningChart,
  synchronizeAxisIntervals,
  translatePoint,
  UIOrigins,
  UIElementBuilders,
  UILayoutBuilders,
  Themes,
} from "@arction/lcjs";

const TIME_DOMAIN = 10 * 1000;
const SAMPLE_RATE = 1000; // points per s

export default {
  name: "ChartData",
  props: ["points"],
  data() {
    this.chart = null;
    return {
      data: null,
      chartId: null,
    };
  },
  methods: {
    initChart(data) {
      const { ecg, bloodPressure, bloodVolume, bloodOxygenation } = data;

      const channels = [
        {
          shortName: "ECG/EKG",
          name: "Electrocardiogram",
          dataSet: ecg,
          color: "#00ff00",
          yStart: -1955,
          yEnd: 1195,
        },
        {
          shortName: "NIBP",
          name: "Blood pressure",
          dataSet: bloodPressure,
          color: undefined,
          yStart: 0.475,
          yEnd: 0.795,
        },
        {
          shortName: "BFV",
          name: "Blood flow volume",
          dataSet: bloodVolume,
          color: undefined,
          yStart: 0.155,
          yEnd: 0.445,
        },
        {
          shortName: "Sp02",
          name: "Blood oxygen saturation",
          dataSet: bloodOxygenation,
          color: undefined,
          yStart: 0.015,
          yEnd: 0.155,
        },
      ];
      const $this = this
      console.debug('chartid:' + $this.chartId)
      this.chart = lightningChart()
        .Dashboard({
          numberOfRows: channels.length,
          numberOfColumns: 1,
          disableAnimations: true,
          container: $this.chartId,
          // theme: Themes.darkGold
        })
        .setRowHeight(0, 0.4)
        .setRowHeight(1, 0.3)
        .setRowHeight(2, 0.2)
        .setRowHeight(3, 0.2);

      console.debug(this.chart)

      const chartList = channels.map((channel, i) => {
        const chart = $this.chart
          .createChartXY({ rowIndex: i, columnIndex: 0 })
          .setPadding({ bottom: 4, top: 4, right: 200, left: 10 })
          .setMouseInteractions(false)
          .setAutoCursorMode(AutoCursorModes.disabled);
        const axisX = chart.getDefaultAxisX().setMouseInteractions(false);
        const axisY = chart
          .getDefaultAxisY()
          .setMouseInteractions(false)
          .setInterval(channel.yStart, channel.yEnd, false, true)
          .setTickStrategy(AxisTickStrategies.Empty)
          .setStrokeStyle(emptyLine);
        if (i > 0) {
          chart.setTitleFillStyle(emptyFill);
        } else {
          chart.setTitle("Medical Dashboard");
        }
        if (i < channels.length - 1) {
          axisX
            .setTickStrategy(AxisTickStrategies.Time, (ticks) =>
              ticks
                .setMajorTickStyle((majorTicks) =>
                  majorTicks
                    .setLabelFillStyle(emptyFill)
                    .setTickStyle(emptyLine)
                    .setTickLength(0)
                    .setTickPadding(0)
                )
                .setMinorTickStyle((minorTicks) =>
                  minorTicks
                    .setLabelFillStyle(emptyFill)
                    .setTickStyle(emptyLine)
                    .setTickLength(0)
                    .setTickPadding(0)
                )
            )
            .setStrokeStyle(emptyLine)
            .setScrollStrategy(undefined);
        } else {
          axisX
            .setTickStrategy(AxisTickStrategies.Time)
            .setInterval(-TIME_DOMAIN, 0)
            .setScrollStrategy(AxisScrollStrategies.progressive);
        }
        return chart;
      });

      const uiList = chartList.map((chart, i) => {
        const axisX = chart.getDefaultAxisX();
        const axisY = chart.getDefaultAxisY();
        const channel = channels[i];
        const ui = chart
          .addUIElement(UILayoutBuilders.Column)
          .setBackground((background) =>
            background.setFillStyle(emptyFill).setStrokeStyle(emptyLine)
          )
          .setMouseInteractions(false)
          .dispose();

        ui.addElement(UIElementBuilders.TextBox).setText(channel.shortName);
        ui.addElement(UIElementBuilders.TextBox)
          .setText(channel.name)
          .setTextFont((font) => font.setSize(10));
        const labelSampleRate = ui
          .addElement(UIElementBuilders.TextBox)
          .setText("")
          .setTextFont((font) => font.setSize(10));

        let labelBpmValue;
        if (channel.name === "Electrocardiogram") {
          const labelBpm = ui
            .addElement(UIElementBuilders.TextBox)
            .setMargin({ top: 10 })
            .setText("BPM");

          labelBpmValue = ui
            .addElement(UIElementBuilders.TextBox)
            .setText("")
            .setTextFont((font) => font.setSize(36));
        }

        requestAnimationFrame(() => {
          ui.restore()
            .setPosition(
              translatePoint(
                { x: axisX.getInterval().end, y: axisY.getInterval().end },
                { x: axisX, y: axisY },
                chart.uiScale
              )
            )
            .setOrigin(UIOrigins.LeftTop)
            .setMargin({ left: 10 });
        });

        return {
          labelSampleRate,
          labelBpmValue,
        };
      });

      synchronizeAxisIntervals(
        ...chartList.map((chart) => chart.getDefaultAxisX())
      );

      const seriesList = chartList.map((chart, i) => {
        const channel = channels[i];
        const series = chart
          .addLineSeries({
            dataPattern: {
              pattern: "ProgressiveX",
            },
          })
          .setName(channel.name)
          .setDataCleaning({ minDataPointCount: 1000 });

        if (channel.color) {
          series.setStrokeStyle((stroke) =>
            stroke.setFillStyle(
              new SolidFill({ color: ColorHEX(channel.color) })
            )
          );
        }
        return series;
      });

      let tSamplePos = window.performance.now();
      let iSampleX = 0;
      const addData = () => {
        const tNow = window.performance.now();
        const seriesNewPoints = seriesList.map((_) => []);
        while (tNow > tSamplePos) {
          const x = tSamplePos;
          for (let i = 0; i < seriesList.length; i += 1) {
            const channel = channels[i];
            const dataSet = channel.dataSet;
            const sample = dataSet[iSampleX % dataSet.length];
            seriesNewPoints[i].push({ x, y: sample });

            if (channel.name === "Electrocardiogram") {
              updateBpm(sample);
            }
          }
          tSamplePos += 1000 / SAMPLE_RATE;
          iSampleX += 1;
        }
        seriesList.forEach((series, i) => series.add(seriesNewPoints[i]));
        channelIncomingDataPointsCount += seriesNewPoints[0].length;
        requestAnimationFrame(addData);
      };
      requestAnimationFrame(addData);

      let channelIncomingDataPointsCount = 0;
      let channelIncomingDataPointsLastUpdate = window.performance.now();
      setInterval(() => {
        const tNow = window.performance.now();
        const chDataPointsPerSecond = Math.round(
          (channelIncomingDataPointsCount * 1000) /
            (tNow - channelIncomingDataPointsLastUpdate)
        );
        const bpm = (beatsCount * 60 * 1000) / (tNow - tStart);

        uiList.forEach((ui, i) => {
          ui.labelSampleRate.setText(
            `${chDataPointsPerSecond} samples / second`
          );
          if (ui.labelBpmValue) {
            ui.labelBpmValue.setText(`${Math.round(bpm)}`);
          }
        });
        channelIncomingDataPointsCount = 0;
        channelIncomingDataPointsLastUpdate = tNow;
      }, 2000);

      const naiveBeatThreshold = 800;
      let tStart = window.performance.now();
      let beatsCount = 0;
      const updateBpm = (() => {
        let lastY = 0;
        return (newSample) => {
          if (lastY < naiveBeatThreshold && newSample > naiveBeatThreshold) {
            // Beat.
            beatsCount += 1;
          }
          lastY = newSample;
        };
      })();
    },
  },
  beforeMount() {
    console.debug('beforeMount')
  },
  mounted() {
    console.debug('Mounted')
    this.chartId = 'id' + Math.trunc(Math.random() * 1000000);
    const $this = this;
    console.debug(this.chartId)
    fetch(document.head.baseURI + "assets/medical-data.json")
      .then((r) => r.json())
      .then((data) => {
        console.info('feteched data')
        $this.initChart(data);
      });
  },
  beforeUnmount() {
    this.chart.dispose();
  },
};
</script>
<style scoped>
.fill {
  height: 100%;
}
</style>

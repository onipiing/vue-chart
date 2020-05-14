<template>
	<div class="charts-container">

		<template v-if="!hasData">
			<p>
				Looks like there is no data with in the selected date/time range or the selected range is invalid.
			</p>
		</template>
		<div class="chart-wrapper" >
			<div class="hpanel">
				<div  class="panel-body">
					<div class="chart">
						<line-chart :id="'chart-'"
									:chart-data='getData2(groupings)'
									:options='groupings.options' />
					</div>
				</div>
			</div>
		</div>
	</div>
</template>

<script>
	import LineChart from './LineChart';
    import {format, isAfter, isBefore} from 'date-fns';
    // import colors from './color-index.json';
    import AppConstants from '../AppConstants.js';
    import jsPDF from 'jspdf';
    import html2canvas from 'html2canvas';
    import {channelData} from '../sample-chart-data.js';
    // import pdfService from '../../../assets/services/PdfService';
    // import alertService from '../../../assets/services/AlertService';
    import * as moment from 'moment';


    import { mapState, mapGetters } from 'vuex';

    export default {
        name: 'dynamic-chart-grouping',
        mounted() {
          this.setupData();
        },
        props: {
            sensorData: {
                type: Array,
                default() {
                  return channelData;
                }
            },
            dateRange: {
                type: Object,
                validation(value) {
                  let isValid = true;
                  if( !value.hasOwnProperty('startDate') && !value.hasOwnProperty('endDate')) {
                    isValid = false;
                  }
                  return isValid;
                }
            },
            maxPoints: {
                type: Number,
                default: AppConstants.MAX_CHART_ITEMS
            }
        },

        data() {
            return {
                hover: false
            };
        },

        computed: {
            ...mapState({
                activeAccount: state => state.session.activeAccount,
                activeDevice: state => state.devices.activeDevice
            }),

            deviceName() {
                return this.activeDevice ? this.activeDevice.deviceName : '';
            },

            deviceId() {
                return this.activeDevice.deviceId;
            },

            accountName() {
                return this.activeAccount ? this.activeAccount.accountName : '';
            },

            styleObject() {
                return {
                    cursor: pointer,
                    color: this.hover ? 'green' : 'black'
                };
            },

            /**
             * Identifies the unique charts based off "unitOfMeasure" and groups readings with the same "unitOfMeasure"
             *
             */
            groupings() {
                const groupings = {};
                const colors = {};
                const options = {
                    scales: {
                        xAxes: [
                            {
                                ticks: {
                                    callback: function (value) {
                                        return format(new Date(value), AppConstants.TABLE_DATE_FORMAT);
                                    },
                                },
                            },
                        ],
                        yAxes: []
                    },
                    maintainAspectRatio: false
                };

                this.sd.forEach((sensor,idx) => {
                    if (sensor['readings'].length === 0) {
                        return;
                    }

                    const unit = ((sensor['unitOfMeasureNameDefault'] === 'user') && (sensor['unitOfMeasureUserDefined'])) ?
                    sensor['unitOfMeasureUserDefined'] :
                    sensor['unitOfMeasureNameDefault'];

                    if (!groupings.hasOwnProperty(unit)) {
                        this.$set(groupings, unit, []);
                        const id = this.getAxisFromUnit(unit);
                        colors[id] = this.getIndexedColor(unit, idx);
                        options['scales']['yAxes'].push({
                            name: unit,
                            type: 'linear',
                            ticks: {
                                beginAtZero: false,
                                fontColor: colors[unit]
                            },
                            gridLines: {
                                display: true,
                                color: colors[unit]
                            },
                            id: id,
                            position: idx % 2 === 0 ? "left" : "right",
                            display: true,
                            label: unit
                        });
                    }
                
                    const displayText = sensor['channelName'] + ' (' + sensor['channelTypeName'] + ')';

                    groupings[unit].push({
                        channelName: sensor['channelName'],
                        channelType: sensor['channelTypeName'],
                        uom: unit,
                        display: displayText,
                        readings: sensor['readings']
                    });

                });

                return {groupings, options, colors};
            }
        },

        methods: {
            setHover(value) {
                this.hover = value === true ? true : false;
            },

            /**
            * Prepares the sensor data to be charted.
            * Filters the data based on the date/time selection.
            */
            setupData() {
                console.log(this.dateRange);
                if (this.dateRange !== undefined) {
                    this.sd = this._filterByDate(this.sensorData)
                } else {
                    this.sd = this._limitDataPoints(this.sensorData);
                }
            },

            _filterByDate(sensorData) {
                return sensorData
                    .map(sd => ({
                        channelName: sd.channelName,
                        channelTypeName: sd.channelTypeName,
                        unitOfMeasureNameDefault: sd.unitOfMeasureNameDefault,
                        unitOfMeasureUserDefined: sd.unitOfMeasureUserDefined,
                        readings: sd['readings']
                            .filter(r => {
                                return (
                                    isAfter(r['readingDate'], this.dateRange.startDate) &&
                                    isBefore(r['readingDate'], this.dateRange.endDate)
                                )
                            })
                        })
                    );
            },

            _limitDataPoints(sensorData) {
                return sensorData.map(sd => ({
                    channelName: sd.channelName,
                    channelTypeName: sd.channelTypeName,
                    unitOfMeasureNameDefault: sd.unitOfMeasureNameDefault,
                    unitOfMeasureUserDefined: sd.unitOfMeasureUserDefined,
                    readings: sd['readings'].slice(0, this.maxPoints)
                }));
            },

            /**
            * Generate the proper "data" object to pass in to ChartJS
            *
            * @param {Object} groupings
            * @returns {{labels: [], datasets: []}}
            */
            getData2(groupings) {
                let  datasets = [];
                let labels = [];
                Object.keys(groupings.groupings).forEach(unit => {
                    datasets = datasets.concat([
                        ...this.getDataSet(groupings.groupings[unit], groupings.colors)
                    ]);
                    labels = this.getLabels(groupings.groupings[unit]);
                });

                if (datasets.length > 0) {
                    this.hasData = true;
                } 
                return { datasets, labels };
            },

            /**
            * Extracts values to be used as labels.
            * Takes care of removing duplicates.
            *
            * @param {Array} channelData
            * @returns {any[]}
            */
            getLabels(channelData) {
                const
                labels = channelData
                    // create an array of arrays with just "readings" data
                    .map(channel => channel['readings'])
                    // now flatten that array of arrays
                    .reduce((acc, cur) => acc.concat(cur),
                    [])
                    // and finally... extract the date from each one
                    .map(r => r['readingDate']);
                // remove duplicates then turn back in to a normal array;
                return Array.from(new Set(labels));
            },

            /**
            * Generate the "datasets" object to pass in to ChartJS
            *
            * @param {Array} channelData List of readings
            * @returns [{label: String, data: []}] Array containing the proper datasets
            */
            getDataSet(channelData, colors) {
                return channelData
                    // create an array of arrays with just "readings" data
                    .map(channel => channel['readings'])
                    .map((reading, index) => {
                        // changed out the display text from groupings
                        //const channelType = channelData[index]['channelType'];
                        const label = channelData[index]['display'];
                        const unit = channelData[index]['uom'];
                        const id = this.getAxisFromUnit(unit);
                        //const color = this.getRandomColor();
                        const colorX = colors[id];

                        return {
                            label,
                            // data: reading.map(r => ({ y: r['readingValue'], x: r['readingDate'] })),
                            data: reading.map(r => ({
                                y: r['readingValue'],
                                x: moment(r['readingDate']).utc().format(AppConstants.TABLE_DATE_FORMAT),
                            })), // demo purpose
                            backgroundColor: colorX,
                            borderColor: colorX,
                            borderWidth: 1,
                            pointRadius: 1,
                            pointBorderWidth: 1,
                            fill: false,
                            id: id,
                        }
                    });
            },

            getRandomColor() {
                var rnum = Math.round(Math.random() * 10);
                var colors = ["#008ed3", "#62aed3", "#009345", "#8bc53f", "#92d2f1", "#40ca81", "#77e0a8", "#d577e0", "#bf752d", "#8a1509"];
                return colors[rnum]
            },

            getIndexedColor(unit, index) {
                var x = Math.round(index % 10);
                var result = "";

                const volts =       ["#0a92ac", "#77cfbe", "#74a56e", "#b7d0a0", "#bc6eb9", "#b494d5", "#2a6d7a", "#81ca9f", "#682714", "#c75c00"];
                const temperature = ["#1a63a6", "#95c3dc", "#2b9220", "#a4db76", "#da0017", "#f78587", "#fc6a09", "#fab25b", "#535353", "#956417"];
                const sensor =      ["#2b6aa9", "#3fa53a", "#843592", "#fc6a0b", "#f367b2", "#a4a4a4", "#7a8dbf", "#946316", "#b19cca", "#a5c3dd"];

                switch (unit.toLowerCase()) {
                    case "volts":
                        result = volts[x];
                        break;
                    case "degc":
                        result = temperature[x];
                        break;
                    default:
                        result = sensor[x];
                }

                //Vue.$log.debug("indexed color found ", index, unit, result);
                return result;
            },
            
            generatePdf(readings, uom) {

                const chartId = 'chart-' + uom;
                const device = this.$store.state.devices.activeDevice;
                const account = this.$store.state.session.activeAccount;

                if (!device || !account) {
                    // alertService.alertError('No device or account found');
                    return;
                }

                const filename = device.deviceName + '_SensorReadings_' + uom + '.pdf';

                // if (device && account && uom) {
                //     document.body.style.cursor = 'wait';
                //     pdfService.createPdf().then(function (pdf) {

                //         var lineY = pdfService.writeDocumentTitleBlock(account, device, pdf, readings, uom);
                //         lineY = pdfService.writeChartImage(pdf, chartId, lineY);

                //         pdfService.writeSummaryTable(pdf, uom, lineY).then(function () {
                //             pdf.save(filename);
                //             document.body.style.cursor = 'default';
                //         });

                //     });
                // }
            },
            // Get yAxis label from unit
            getAxisFromUnit(unit) {
                return unit.replace(' ', '_');
            }
            
    },

    watch: {
        sensorData() {
            this.setupData();
        },
        dateRange() {
            this.setupData();
        }
    },
    data() {
        return {
            sd: [],
            hasData: false,
            pdfDocument: {}
        }
    },
    components: {
      LineChart,
    }
  }
</script>

<style scoped>
	.charts-container {
		display: flex;
		flex-direction: column;
	}

	.chart-wrapper {
		width: 100%;
		/*max-height: 400px;
		margin-bottom: 50px;*/
	}

	.chart {
		width: 100%;
		height: 400px;
		border: 1px solid lightgray;
	}

	@media (max-width: 100%) {
		body {
			text-align: center;
		}

		.charts-container {
			flex-direction: column;
		}

		.chart-wrapper {
			width: 100%;
		}
	}

    i.linkActive {
        color: green;
    }
	div.linkActive {
		cursor: pointer;
	}

</style>
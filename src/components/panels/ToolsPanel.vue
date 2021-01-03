<style scoped>
table {
	width: 100%;
	border-spacing: 0;
}

table td,
table th {
	text-align: center;
}

table.tools th,
table.tools td {
	width: 20%;
}

table.extra th,
table.extra td {
	width: 25%;
}

table.extra tr > th:first-child,
table.extra tr > td:first-child {
	width: 50%;
}
</style>

<template>
	<v-card>
		<v-card-title class="py-2">
			<panel-link :active="currentPage !== 'tools'" @click="currentPage = 'tools'" class="mr-2">
				<v-icon small>mdi-wrench</v-icon> {{ $t('panel.tools.caption') }}
			</panel-link>
			<v-spacer></v-spacer>
		</v-card-title>

		<v-card-text class="pa-0">
			<template v-if="currentPage === 'tools'">
				<table class="tools" v-show="canShowTools">
					<thead>
						<th class="pl-1">{{ $t('panel.tools.tool', ['']) }}</th>
						<th class="pl-1">{{ $t('panel.tools.name', ['']) }}</th>
						<th class="px-1">{{ $t('panel.tools.current', ['']) }}</th>
						<th class="px-1">{{ $t('panel.tools.active') }}</th>
						<th class="pr-2">Offsets</th>
					</thead>
					<tbody>
						<!-- Tools -->
						<template v-for="(tool, toolIndex) in visibleTools">
							<!-- Tool -->
							<tr v-for="(toolHeater, toolHeaterIndex) in getToolHeaters(tool)" :key="`tool-${toolIndex}-${toolHeaterIndex}`" :class="{ [selectedToolClass] : (tool.number === state.currentTool) }">
								
								<th v-if="toolHeaterIndex === 0" :rowspan="Math.max(1, tool.heaters.length)" class="pl-1" :class="{ 'pt-2 pb-2' : !tool.heaters.length && !toolHeater }">
									<span class="font-weight-regular caption"> T{{ tool.number }} </span>
								</th>

								<th v-if="toolHeaterIndex === 0" :rowspan="Math.max(1, tool.heaters.length)" class="pl-1" :class="{ 'pt-2 pb-2' : !tool.heaters.length && !toolHeater }">
									<a href="javascript:void(0)" @click="toolClick(tool)"> {{ tool.name || $t('panel.tools.tool', [tool.number]) }} </a>
								</th>

								<template v-if="!toolHeater && getSpindle(tool)">
									<td class="text-center"> {{ $display(getSpindle(tool).current, 0, $t('generic.rpm')) }} </td>
									<td> <tool-input :spindle="getSpindle(tool)" :spindle-index="getSpindleIndex(tool)" active></tool-input> </td>
									<td> {{ tool.offsets[0] }}, {{ tool.offsets[1]}}, {{ tool.offsets[2] }} </td>
								</template>

								<template v-else>
									<td> {{ getHeaterValue(toolHeater) }} </td>
									<td class="pl-2 pr-1"> {{ getHeaterValue(toolHeater) }} </td>
									<td> {{ tool.offsets[0] }}, {{ tool.offsets[1]}}, {{ tool.offsets[2] }} </td>
								</template>
							</tr>

							<!-- Divider -->
							<tr v-if="toolIndex < visibleTools.length - 1" :key="`div-tool-${toolIndex}`">
								<td colspan="5">
									<v-divider></v-divider>
								</td>
							</tr>
						</template>


					</tbody>
				</table>

				<v-alert :value="!canShowTools" type="info" class="mb-0"> {{ $t('panel.tools.noTools') }} </v-alert>

			</template>

		</v-card-text>
	</v-card>
</template>

<script>
'use strict'

import { mapState, mapGetters, mapActions, mapMutations } from 'vuex'

import { AnalogSensorType, HeaterState } from '../../store/machine/modelEnums.js'
import { getHeaterColor, getExtraColor } from '../../utils/colors.js'
import { DisconnectedError } from '../../utils/errors.js'

export default {
	computed: {
		...mapGetters(['isConnected', 'uiFrozen']),
		...mapState('machine/model', ['heat', 'move', 'sensors', 'state', 'spindles', 'tools']),
		...mapState('machine/settings', ['displayedExtraTemperatures']),
		...mapState('settings', ['darkTheme']),
		canTurnEverythingOff() {
			return (!this.uiFrozen &&
					(this.tools.some(tool => tool && tool.heaters.some(toolHeater => toolHeater >= 0 && toolHeater < this.heat.heaters.length &&
						this.heat.heaters[toolHeater] && this.heat.heaters[toolHeater].state !== HeaterState.off, this), this) ||
					this.heat.bedHeaters.some(bedHeater => bedHeater >= 0 && bedHeater < this.heat.heaters.length &&
						this.heat.heaters[bedHeater] && this.heat.heaters[bedHeater].state !== HeaterState.off, this) ||
					this.heat.chamberHeaters.some(chamberHeater => chamberHeater >= 0 && chamberHeater < this.heat.heaters.length &&
						this.heat.heaters[chamberHeater] && this.heat.heaters[chamberHeater].state !== HeaterState.off, this)));
		},
		visibleTools() {
			return this.tools.filter(tool => tool !== null);
		},
		canShowTools() {
			return (this.visibleTools.length > 0 ||
					this.bedHeaters.some(bed => bed !== null) ||
					this.chamberHeaters.some(chamber => chamber !== null));
		},
		selectedToolClass() {
			return this.darkTheme ? 'grey darken-3' : 'blue lighten-5';
		},
		extraSensors() {
			return this.sensors.analog
				.map((sensor, index) => ({ sensor, index }))
				.filter(item => item.sensor && !this.heat.heaters.some(heater => heater && heater.sensor === item.index));
		},
		bedHeaters() {
			return this.heat.bedHeaters
				.map(heaterIndex => {
					if (heaterIndex >= 0 && heaterIndex < this.heat.heaters.length && this.heat.heaters[heaterIndex]) {
						return this.heat.heaters[heaterIndex];
					}
					return null;
				});
		},
		hasOneBed() {
			return this.bedHeaters.filter(bed => bed).length === 1;
		},
		chamberHeaters() {
			return this.heat.chamberHeaters
				.map(heaterIndex => {
					if (heaterIndex >= 0 && heaterIndex < this.heat.heaters.length && this.heat.heaters[heaterIndex]) {
						return this.heat.heaters[heaterIndex];
					}
					return null;
				});
		},
		hasOneChamber() {
			return this.chamberHeaters.filter(chamber => chamber).length === 1;
		}
	},
	data() {
		return {
			dropdownShown: false,
			turningEverythingOff: false,

			currentPage: 'tools',
			waitingForCode: false,

			loadingFilament: false,
			filamentMenu: {
				tool: undefined,
				dialogShown: false
			},

			resetHeaterFault: false,
			faultyHeater: -1
		}
	},
	methods: {
		...mapActions('machine', ['sendCode']),
		...mapMutations('machine/settings', ['toggleExtraVisibility']),

		// Control All Drop-Down
		showDropdown() {
			this.dropdownShown = !this.dropdownShown;
			if (this.dropdownShown) {
				const input = this.$refs.allActive;
				setTimeout(() => input.focus(), 300);
			}
		},
		async turnEverythingOff() {
			let code = '';
			this.tools.forEach(function(tool) {
				if (tool && tool.heaters.length) {
					const temps = tool.heaters.map(() => '-273.15').join(':');
					code += `G10 P${tool.number} R${temps} S${temps}\n`;
				}
			});
			this.heat.bedHeaters.forEach(function(bedHeater, index) {
				if (bedHeater >= -1 && bedHeater < this.heat.heaters.length) {
					code += `M140 P${index} S-273.15\n`;
				}
			}, this);
			this.heat.chamberHeaters.forEach(function(chamberHeater, index) {
				if (chamberHeater >= -1 && chamberHeater < this.heat.heaters.length) {
					code += `M141 P${index} S-273.15\n`;
				}
			}, this);

			this.turningEverythingOff = true;
			try {
				await this.sendCode(code);
			} catch (e) {
				if (!(e instanceof DisconnectedError)) {
					this.$log('error', this.$t('error.turnOffEverythingFailed'), e.message);
				}
			}
			this.turningEverythingOff = false;
		},

		// Heaters
		getHeaterColor: heater => getHeaterColor(heater),
		getHeaterName(heater, heaterIndex) {
			if (heater && heater.sensor >= 0 && heater.sensor < this.sensors.analog.length) {
				const sensor = this.sensors.analog[heater.sensor];
				if (sensor && sensor.name) {
					const matches = /(.*)\[(.*)\]$/.exec(sensor.name);
					if (matches) {
						return matches[1];
					}
					return sensor.name;
				}
			}
			return this.$t('panel.tools.heater', [heaterIndex]);
		},
		getHeaterValue(heater) {
			if (heater && heater.sensor >= 0 && heater.sensor < this.sensors.analog.length) {
				const sensor = this.sensors.analog[heater.sensor];
				if (sensor) {
					return this.formatSensorValue(sensor);
				}
			}
			return this.$t('generic.noValue');
		},
		formatSensorValue(sensor) {
			if (sensor.name) {
				const matches = /(.*)\[(.*)\]$/.exec(sensor.name);
				if (matches) {
					return this.$display(sensor.lastReading, 1, matches[2]);
				}
			}
			const unit = (sensor.type === AnalogSensorType.dhtHumidity) ? '%RH' : 'C';
			return this.$display(sensor.lastReading, 1, unit);
		},

		// Tools
		toolClick(tool) {
			if (!this.isConnected) {
				return;
			}

			this.waitingForCode = true;
			try {
				if (this.state.currentTool === tool.number) {
					// Deselect current tool
					this.sendCode('T-1');
				} else {
					// Select new tool
					this.sendCode(`T${tool.number}`);
				}
			} catch (e) {
				if (!(e instanceof DisconnectedError)) {
					this.$log('error', e.message);
				}
			}
			this.waitingForCode = false;
		},
		getFilament(tool) {
			if (this.isNumber(tool.filamentExtruder) && tool.filamentExtruder >= 0 && tool.filamentExtruder < this.move.extruders.length) {
				return this.move.extruders[tool.filamentExtruder].filament;
			}
			return null;
		},
		canLoadFilament(tool) {
			return !this.uiFrozen && tool.filamentExtruder >= 0 && tool.filamentExtruder < this.move.extruders.length;
		},
		unloadFilament() {
			if (!this.isConnected) {
				return;
			}

			let code = '';
			if (this.state.currentTool !== this.filamentMenu.tool.number) {
				code = `T${this.filamentMenu.tool.number}\n`;
			}
			code += 'M702';
			this.sendCode(code);
		},
		
		getToolHeaters(tool) {
			const toolHeaters = tool.heaters
				.filter(heaterIndex => heaterIndex >= 0 && heaterIndex < this.heat.heaters.length && this.heat.heaters[heaterIndex], this)
				.map(heaterIndex => this.heat.heaters[heaterIndex], this);
			return toolHeaters.length ? toolHeaters : [null];
		},
		toolHeaterClick(tool, heater) {
			if (!this.isConnected) {
				return;
			}

			let offTemps;
			switch (heater.state) {
				case HeaterState.off:		// Off -> Active
					this.sendCode(`T${tool.number}`);
					break;

				case HeaterState.standby:	// Standby -> Off
					offTemps = tool.active.map(() => '-273.15').join(':');
					this.sendCode(`G10 P${tool.number} S${offTemps} R${offTemps}`);
					break;

				case HeaterState.active:	// Active -> Standby
					this.sendCode('T-1');
					break;

				case HeaterState.fault:		// Fault -> Ask for reset
					this.faultyHeater = this.heat.heaters.indexOf(heater);
					this.resetHeaterFault = true;
					break;
			}
		},

		getSpindle(tool) {
			return this.spindles.find(spindle => spindle.tool === tool.number);
		},
		getSpindleIndex(tool) {
			return this.spindles.findIndex(spindle => spindle.tool === tool.number);
		},

		// Beds
		bedHeaterClick(bedHeater, bedIndex) {
			if (!this.isConnected) {
				return;
			}

			switch (bedHeater.state) {
				case HeaterState.off:		// Off -> Active
					this.sendCode(`M140 P${bedIndex} S${bedHeater.active}`);
					break;

				case HeaterState.standby:	// Standby -> Off
					this.sendCode(`M140 P${bedIndex} S-273.15`);
					break;

				case HeaterState.active:	// Active -> Standby
					this.sendCode(`M144 P${bedIndex}`);
					break;

				case HeaterState.fault:		// Fault -> Ask for reset
					this.faultyHeater = this.heat.heaters.indexOf(bedHeater);
					this.resetHeaterFault = true;
					break;
			}
		},

		// Chambers
		chamberHeaterClick(chamberHeater, chamberIndex) {
			if (!this.isConnected) {
				return;
			}

			switch (chamberHeater.state) {
				case HeaterState.off:		// Off -> Active
					this.sendCode(`M141 P${chamberIndex} S${chamberHeater.active}`);
					break;

				// Standby mode for chambers is not officially supported yet (there's no code for standby control)

				case HeaterState.fault:		// Fault -> Ask for reset
					this.faultyHeater = this.heat.heaters.indexOf(chamberHeater);
					this.resetHeaterFault = true;
					break;

				default:	// Active -> Off
					this.sendCode(`M141 P${chamberIndex} S-273.15`);
					break;
			}
		},

		// Extra
		getExtraColor: item => getExtraColor(item),
		formatExtraName(item) {
			if (item.sensor.name) {
				const matches = /(.*)\[(.*)\]$/.exec(item.sensor.name);
				if (matches) {
					return matches[1];
				}
				return item.sensor.name;
			}
			return this.$t('panel.tools.extra.sensorIndex', [item.index]);
		}
	}
}
</script>

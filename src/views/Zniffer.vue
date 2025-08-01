<template>
	<v-container class="px-3" style="max-width: unset">
		<multipane class="horizontal-panes" layout="horizontal">
			<div
				class="pane"
				ref="topPane"
				v-intersect.once="bindTopPaneObserver"
				:style="{
					minHeight: '300px',
					height: `${topPaneHeight}px`,
				}"
			>
				<v-row v-if="zniffer.enabled">
					<v-col style="max-width: 280px; margin-top: 7px">
						<v-btn-group density="compact" multiple>
							<v-btn
								size="small"
								v-for="button in buttons"
								:key="button.id"
								:id="button.id"
								:color="button.disabled ? 'grey' : button.color"
								:disabled="button.disabled"
								@click="button.action"
								v-tooltip:bottom="button.tooltip"
							>
								<v-icon>{{ button.icon }}</v-icon>
							</v-btn>
						</v-btn-group>
					</v-col>
					<v-col
						ref="settingCol"
						v-resize="onTopColResize"
						class="pa-0 pt-2"
						cols="6"
					>
						<v-text-field
							v-model="search"
							clearable
							flat
							persistent-hint
							hint="Search expression using JS. Click on info button for more info"
							:error="searchError"
							:error-messages="
								searchError ? ['Invalid search'] : []
							"
							variant="outlined"
							density="compact"
							single-line
							class="ma-2"
							prepend-inner-icon="search"
							label="Search"
						>
							<template #append>
								<v-menu location="bottom">
									<template #activator="{ props }">
										<v-icon
											v-bind="props"
											color="grey"
											size="20"
											>info</v-icon
										>
									</template>
									<v-card>
										<v-card-text>
											<p>
												Write a custom filter function
												in JS. Function arguments are:
												frame (the full frame object),
												homeId, ch, src, dest,
												protocolDataRate, hop, dir
												(direction), repeaters.
											</p>
											<strong>Examples:</strong>
											<ul>
												<li>
													<code>frame.corrupted</code>
												</li>
												<li>
													<code
														>src === 1 && dest ===
														2</code
													>
												</li>
												<li>
													<code
														>protocolDataRate ===
														'100kbps'</code
													>
												</li>
												<li>
													<code
														>homeId ===
														'12345678'</code
													>
												</li>
												<li>
													<code>hop > 1</code>
												</li>

												<li>
													<code
														>dir === 'inbound'</code
													>
												</li>
												<li>
													<code
														>repeaters.length >
														0</code
													>
												</li>
											</ul>
										</v-card-text>
									</v-card>
								</v-menu>
								<v-col
									v-if="totalFrames"
									style="margin-top: -7px"
									class="pa-0 ml-3 text-caption text-grey text-center"
								>
									<p class="mb-0">Frames</p>
									<p class="mb-0">
										{{
											humanFriendlyNumber(totalFrames, 2)
										}}
									</p>
								</v-col>
							</template>
							<!-- add to append slot the total numer of frames -->
						</v-text-field>
					</v-col>

					<v-col cols="12">
						<v-data-table
							v-intersect.once="bindScroll"
							:headers="headers"
							:items="framesLimited"
							:row-props="getRowStyle"
							@click:row="onRowClick"
							show-select
							fixed-header
							density="compact"
							:height="topPaneHeight - offsetTop"
							id="framesTable"
							ref="framesTable"
							hide-default-footer
							disable-pagination
							:mobile-breakpoint="-1"
						>
							<template #[`item.timestamp`]="{ item }">
								<span
									v-tooltip:bottom="
										new Date(
											item.timestamp,
										).toLocaleDateString()
									"
								>
									{{ getTimestamp(item.timestamp) }}
								</span>
							</template>

							<template #[`item.channel`]="{ item }">
								{{ item.channel }}
							</template>

							<template #[`item.rssi`]="{ item }">
								{{ getRssi(item) }}
							</template>

							<template #[`item.protocolDataRate`]="{ item }">
								<span v-if="item.corrupted"></span>
								<div v-else class="d-flex text-center">
									<rich-value
										:value="getProtocolIcon(item.protocol)"
									/>
									<span class="ml-1">
										{{
											getProtocolDataRate(item, false) +
											(item.speedModified ? ' 🐌' : '')
										}}
									</span>
								</div>
							</template>

							<template #[`item.type`]="{ item }">
								<span>
									{{ getType(item) }}
								</span>
							</template>

							<template #[`item.sourceNodeId`]="{ item }">
								<span v-html="getRoute(item)"></span>
							</template>

							<template #[`item.homeId`]="{ item }">
								{{ item.homeId?.toString(16) }}
							</template>

							<template #[`item.payload`]="{ item }">
								<code v-if="item.corrupted"> CRC Error </code>
								<span v-else-if="item.parsedPayload">
									{{ getPayloadTags(item.parsedPayload) }}
								</span>
								<span
									v-else-if="item.payload && !item.corrupted"
								>
									{{ item.payload }}
								</span>
								<span v-else-if="item.routedAck"
									>ROUTED ACK</span
								>
								<span v-else>---</span>
							</template>

							<template v-if="start > 0" #[`body.prepend`]>
								<tr>
									<td
										:colspan="headers.length"
										class="text-center"
										:style="
											'padding-top:' + startHeight + 'px'
										"
									>
										<!-- <v-skeleton-loader
											type="table-row"
										/><v-skeleton-loader type="table-row" /> -->
									</td>
								</tr>
							</template>
							<template
								v-if="start + perPage <= totalFrames"
								#[`body.append`]
							>
								<tr>
									<td
										:colspan="headers.length"
										class="text-center"
										:style="
											'padding-top:' + endHeight + 'px'
										"
									>
										<!-- <v-skeleton-loader type="table-row" /> -->
									</td>
								</tr>
							</template>
						</v-data-table>

						<v-fab
							v-if="!autoScroll"
							color="purple"
							@click="enableAutoScroll()"
							size="small"
							hover
							icon
							location="top right"
							absolute
							style="top: 40px"
							v-tooltip:bottom="'Enable autoscroll'"
						>
							<v-icon>vertical_align_bottom</v-icon>
						</v-fab>
					</v-col>
				</v-row>
				<v-row v-else>
					<v-col cols="12">
						<v-alert text type="info">
							<span
								>Zniffer is disabled, enable it in
								settings</span
							>
						</v-alert>
					</v-col>
				</v-row>
			</div>
			<multipane-resizer
				:class="$vuetify.theme.current.dark ? 'dark' : ''"
			/>
			<div
				class="pane pa-2"
				style="flex-grow: 1; overflow-y: scroll; overflow-x: hidden"
				:style="{
					height: `calc(100vh - ${topPaneHeight + 110}px)`,
				}"
			>
				<frame-details class="my-1" :modelValue="selectedFrame" />
			</div>
		</multipane>

		<v-fab
			color="primary"
			@click="drawer = !drawer"
			hover
			app
			icon
			size="large"
			location="bottom right"
		>
			<v-icon v-if="drawer">close</v-icon>
			<v-icon v-else>menu</v-icon>
		</v-fab>

		<v-navigation-drawer
			v-model="drawer"
			absolute
			location="right"
			style="z-index: 2"
		>
			<v-card class="fill">
				<v-card-title> Settings </v-card-title>
				<v-card-text>
					<v-row>
						<v-col v-if="!isPopup" cols="12">
							<v-btn
								variant="text"
								size="small"
								color="primary"
								@click="openInWindow('Zniffer', 800, 1500)"
							>
								Open in new<v-icon>open_in_new</v-icon>
							</v-btn>
						</v-col>
						<v-col cols="12">
							<v-select
								label="Zniffer frequency"
								hide-details
								:items="znifferRegions"
								v-model="frequency"
								@update:model-value="setFrequency"
							>
								<template #prepend>
									<v-icon>signal_cellular_alt</v-icon>
								</template>

								<template #append>
									<v-icon
										color="success"
										v-if="frequencySuccess"
										>check_circle</v-icon
									>
								</template>
							</v-select>
						</v-col>
						<v-col cols="12" v-if="lrChannelConfigAvailable">
							<v-select
								label="LR Channel Configuration"
								hide-details
								:items="znifferLRChannelConfigs"
								v-model="lrChannelConfig"
								@update:model-value="setLRChannelConfig"
							>
								<template #prepend>
									<v-icon>wifi_channel</v-icon>
								</template>

								<template #append>
									<v-icon
										color="success"
										v-if="lrChannelConfigSuccess"
										>check_circle</v-icon
									>
								</template>
							</v-select>
						</v-col>
					</v-row>
				</v-card-text>
			</v-card>
		</v-navigation-drawer>
	</v-container>
</template>
<script>
import { defineAsyncComponent, nextTick } from 'vue'
import { ZWaveFrameType, LongRangeFrameType } from 'zwave-js'
import { Protocols, RFRegion } from '@zwave-js/core'
import { socketEvents } from '@server/lib/SocketEvents'

import { mapState, mapActions } from 'pinia'
import useBaseStore from '../stores/base.js'
import { inboundEvents as socketActions } from '@server/lib/SocketEvents'
import InstancesMixin from '../mixins/InstancesMixin.js'
import {
	znifferRegions as defaultZnifferRegions,
	znifferLRChannelConfigs as defaultZnifferLRChannelConfigs,
} from '../lib/items.js'
import {
	getRoute,
	getType,
	getRssi,
	getProtocolDataRate,
	humanFriendlyNumber,
	openInWindow,
	getProtocolIcon,
	isPopupWindow,
} from '../lib/utils'
import { instances, manager } from '../lib/instanceManager.js'

export default {
	name: 'Zniffer',
	mixins: [InstancesMixin],
	props: {
		socket: Object,
	},
	components: {
		Multipane: defineAsyncComponent(
			() => import('../components/custom/Multipane.vue'),
		),
		MultipaneResizer: defineAsyncComponent(
			() => import('../components/custom/MultipaneResizer.vue'),
		),
		RichValue: defineAsyncComponent(
			() => import('@/components/nodes-table/RichValue.vue'),
		),
		FrameDetails: defineAsyncComponent(
			() => import('../components//custom/FrameDetails.vue'),
		),
	},
	computed: {
		...mapState(useBaseStore, ['zniffer', 'znifferState']),
		app() {
			return manager.getInstance(instances.APP)
		},
		buttons() {
			return [
				{
					id: 'start',
					icon: 'play_arrow',
					color: 'success',
					tooltip: 'Start Zniffer',
					action: this.startZniffer,
					disabled: this.viewState === 'recording',
				},
				{
					id: 'stop',
					icon: 'stop',
					color: 'error',
					tooltip: 'Stop Zniffer',
					action: this.stopZniffer,
					disabled: this.viewState !== 'recording',
				},
				{
					id: 'clear',
					icon: 'delete',
					color: 'warning',
					tooltip: 'Clear Zniffer',
					action: this.clearFrames,
					disabled:
						this.frames.length === 0 || this.viewState === 'loaded',
				},
				{
					id: 'load',
					icon: 'folder_open',
					color: 'info',
					tooltip: 'Load capture from file',
					action: this.loadCapture,
					disabled: this.viewState === 'recording',
				},
				{
					id: 'save',
					icon: 'save',
					color: 'primary',
					tooltip: 'Save capture',
					action: this.createCapture,
					disabled:
						this.frames.length === 0 || this.viewState === 'loaded',
				},
			]
		},
		framesLimited() {
			return this.framesFiltered.slice(
				this.start,
				this.perPage + this.start,
			)
		},
		totalFrames() {
			return this.framesFiltered.length
		},
		startHeight() {
			return this.start * this.rowHeight
		},
		endHeight() {
			const lastIndex = this.start + this.perPage
			return this.rowHeight * (this.totalFrames - lastIndex + 1)
		},
	},
	watch: {
		topPaneHeight() {
			this.resizeScrollWrapper()
		},
		znifferState() {
			this.clearZnifferConfiguration()
			// Sync the view state with the zniffer state
			if (this.znifferState?.started && this.viewState !== 'recording') {
				this.viewState = 'recording'
			} else if (this.viewState === 'recording') {
				this.viewState = 'stopped'
			}
		},
		frequencySuccess(v) {
			if (v) {
				setTimeout(() => {
					this.frequencySuccess = false
				}, 3000)
			}
		},
		lrChannelConfigSuccess(v) {
			if (v) {
				setTimeout(() => {
					this.lrChannelConfigSuccess = false
				}, 3000)
			}
		},
		framesQueue: {
			deep: 1,
			handler() {
				// used to improve performances when lot of frames comes all together
				if (this.queueTimeout) {
					clearTimeout(this.queueTimeout)
				}

				this.queueTimeout = setTimeout(() => {
					this.emptyQueue()
				}, 50)
			},
		},
		search(v) {
			if (this.searchTimeout) {
				clearTimeout(this.searchTimeout)
			}

			this.searchTimeout = setTimeout(() => {
				this.filterFrames(v)
			}, 300)
		},
		frames: {
			deep: 1,
			handler() {
				this.filterFrames(this.search)
				this.scrollBottom()

				if (this.viewState === 'initial' && this.frames.length > 0) {
					// If we were in initial state and suddenly have frames,
					// the Zniffer might be active
					if (this.znifferState?.started) {
						this.viewState = 'recording'
					} else {
						this.viewState = 'stopped'
					}
				}
			},
		},
	},
	mounted() {
		this.socket.on(socketEvents.znifferFrame, this.addFrame)

		this.onWindowResize = () => {
			this.topPaneHeight = window.innerHeight / 2
		}

		this.socket.on('connect', this.onConnnect)

		if (this.socket.connected) {
			this.onConnnect()
		}

		window.addEventListener('resize', this.onWindowResize)

		this.onWindowResize()
		this.clearZnifferConfiguration()
		this.initializeViewState()
	},
	beforeUnmount() {
		window.removeEventListener('resize', this.onWindowResize)

		if (this.roTopPane) {
			this.roTopPane.unobserve(this.$refs.topPane)
			this.roTopPane.disconnect()
		}

		if (this.socket) {
			// unbind events
			this.socket.off(socketEvents.znifferFrame)
			this.socket.off('connect', this.onConnnect)
		}

		if (this.timeoutScroll) {
			clearTimeout(this.timeoutScroll)
		}

		if (this.searchTimeout) {
			clearTimeout(this.searchTimeout)
		}
	},
	data() {
		return {
			znifferRegions: defaultZnifferRegions,
			znifferLRChannelConfigs: defaultZnifferLRChannelConfigs,
			isPopup: isPopupWindow(),
			fab: false,
			drawer: false,
			frequency: null,
			frequencySuccess: false,
			lrRegions: [],
			lrChannelConfigAvailable: false,
			lrChannelConfig: null,
			lrChannelConfigSuccess: false,
			start: 0,
			offsetTop: 125,
			search: '',
			searchError: false,
			viewState: 'initial', // 'initial', 'recording', 'stopped', 'loaded'
			busy: false,
			selectedFrame: null,
			scrollWrapper: null,
			rowHeight: 41,
			perPage: 22,
			topPaneHeight: 500,
			frames: [],
			framesFiltered: [],
			framesQueue: [],
			headers: [
				{
					title: '#',
					key: 'id',
					width: '10ch',
					sortable: false,
				},
				{
					title: 'Timestamp',
					key: 'timestamp',
					width: '12ch',
					sortable: false,
					class: 'no-wrap',
				},
				{
					title: 'Delta [ms]',
					key: 'delta',
					width: '4ch',
					sortable: false,
				},
				{
					title: 'Protocol Data Rate',
					key: 'protocolDataRate',
					width: '20ch',
					sortable: false,
				},
				{
					title: 'RSSI',
					key: 'rssi',
					width: '4ch',
					sortable: false,
				},
				{
					title: 'Ch',
					key: 'channel',
					width: '4ch',
					sortable: false,
				},
				{
					title: 'Home Id',
					key: 'homeId',
					width: '8ch',
					sortable: false,
					class: 'no-wrap',
				},
				{
					title: 'Type',
					width: '8ch',
					key: 'type',
					sortable: false,
				},
				{
					title: 'Route',
					key: 'sourceNodeId',
					sortable: false,
				},
				{
					title: 'Payload',
					key: 'payload',
					sortable: false,
				},
			],
			autoScroll: true,
		}
	},
	methods: {
		...mapActions(useBaseStore, ['showSnackbar']),
		openInWindow,
		humanFriendlyNumber,
		initializeViewState() {
			// Verify that the initial view state is set correctly
			if (this.znifferState?.started) {
				this.viewState = 'recording'
			} else if (this.frames.length > 0) {
				this.viewState = 'stopped'
			} else {
				this.viewState = 'initial'
			}
		},
		getProtocolIcon(protocol) {
			return getProtocolIcon(protocol, this.app.currentTheme)
		},
		emptyQueue() {
			if (this.framesQueue.length > 0) {
				this.frames.push(...this.framesQueue)
				this.framesQueue = []
			}
		},
		async onConnnect() {
			await this.getFrames()

			// Try to restore the correct initial state
			if (this.viewState === 'initial' && this.frames.length > 0) {
				this.viewState = this.znifferState?.started
					? 'recording'
					: 'stopped'
			}
		},
		addFrame(data) {
			const lastFrame =
				this.framesQueue.length > 0
					? this.framesQueue[this.framesQueue.length - 1]
					: this.frames[this.frames.length - 1]

			data.id = lastFrame ? lastFrame.id + 1 : 1
			data.delta = lastFrame ? data.timestamp - lastFrame.timestamp : 0
			this.framesQueue.push(data)
		},
		async clearZnifferConfiguration() {
			// needed to handle the clear event on select
			await nextTick()

			if (this.znifferState) {
				this.znifferRegions = Object.entries(
					this.znifferState.supportedFrequencies,
				)
					.map(([key, value]) => {
						const region = parseInt(key, 10)
						return {
							title: value,
							value: region,
							disabled:
								region === RFRegion.Unknown ||
								region === RFRegion['Default (EU)'],
						}
					})
					.sort((a, b) => a.title?.localeCompare(b.title))
				this.frequency = this.znifferState.frequency
				this.lrRegions = this.znifferState.lrRegions
				this.znifferLRChannelConfigs = Object.entries(
					this.znifferState.supportedLRChannelConfigs,
				).map(([key, value]) => {
					return {
						title: value,
						value: parseInt(key, 10),
					}
				})
				this.lrChannelConfig = this.znifferState.lrChannelConfig
				this.lrChannelConfigAvailable = this.lrRegions.includes(
					this.frequency,
				)
			} else {
				this.znifferRegions = defaultZnifferRegions
				this.frequency = null
				this.lrRegions = []
				this.znifferLRChannelConfigs = defaultZnifferLRChannelConfigs
				this.lrChannelConfig = null
				this.lrChannelConfigAvailable = false
			}
		},
		onTopColResize() {
			this.offsetTop = this.$refs.settingCol.$el.clientHeight + 20
			this.resizeScrollWrapper()
		},
		resizeScrollWrapper() {
			if (this.scrollWrapper) {
				this.scrollWrapper.style.height = `${
					this.topPaneHeight - this.offsetTop
				}px`
			}
		},
		filterFrames(search) {
			if (!search || search.trim() === '') {
				this.framesFiltered = this.frames
				this.searchError = false
				return
			}

			try {
				// sanitize search function, convert assignment to comparison
				search = search.replace(/([^=])=([^=])/g, '$1==$2')
				const fn = new Function(
					'frame, homeId, ch, src, dest, protocolDataRate, hop, dir, repeaters',
					`return ${search.replace(/\\/g, '\\\\')}`,
				)

				this.framesFiltered = this.frames.filter((frame) => {
					return fn(
						frame,
						frame.homeId?.toString(16) || '',
						frame.channel,
						frame.sourceNodeId,
						frame.destinationNodeId,
						frame.protocolDataRate,
						frame.hop,
						frame.direction,
						frame.repeaters || [],
					)
				})

				this.searchError = false
			} catch (e) {
				this.searchError = true
			}
		},
		onRowClick(
			event,
			{ item, toggleSelect, internalItem, index, isSelected },
		) {
			if (isSelected(item)) {
				this.selectedFrame = null
				this.autoScroll = true
			} else {
				this.selectedFrame = item
				this.autoScroll = false
			}
			toggleSelect(internalItem, index, event)
		},
		getRowStyle({ item: frame }) {
			const style = {
				backgroundColor: '',
			}

			if (this.selectedFrame && this.selectedFrame.id === frame.id) {
				style.backgroundColor = 'rgba(0, 0, 255, 0.5)'
			} else if (frame.corrupted) {
				style.backgroundColor = 'rgba(255, 0, 0, 0.1)'
			} else if (frame.routedAck) {
				style.backgroundColor = 'rgba(255, 165, 0, 0.1)'
			} else {
				if (frame.protocol === Protocols.ZWaveLongRange) {
					switch (frame.type) {
						case LongRangeFrameType.Singlecast:
							style.backgroundColor = 'rgba(0, 255, 0, 0.1)'
							break
						case LongRangeFrameType.Ack:
							style.backgroundColor = 'rgba(255, 165, 0, 0.1)'
							break
						case LongRangeFrameType.BeamStart:
							style.backgroundColor = 'rgba(255, 255, 255, 0.1)'
							break
						case LongRangeFrameType.BeamStop:
							style.backgroundColor = 'rgba(0, 0, 0, 0.1)'
							break
						case LongRangeFrameType.Broadcast:
							style.backgroundColor = 'rgba(165, 0, 255, 0.1)'
							break
					}
				} else {
					switch (frame.type) {
						case ZWaveFrameType.Singlecast:
							style.backgroundColor = 'rgba(0, 255, 0, 0.1)'
							break
						case ZWaveFrameType.Multicast:
							style.backgroundColor = 'rgba(0, 0, 255, 0.1)'
							break
						case ZWaveFrameType.AckDirect:
							style.backgroundColor = 'rgba(255, 165, 0, 0.1)'
							break
						case ZWaveFrameType.ExplorerNormal:
							style.backgroundColor = 'rgba(255, 255, 0, 0.1)'
							break
						case ZWaveFrameType.ExplorerSearchResult:
							style.backgroundColor = 'rgba(255, 0, 255, 0.1)'
							break
						case ZWaveFrameType.ExplorerInclusionRequest:
							style.backgroundColor = 'rgba(0, 255, 255, 0.1)'
							break
						case ZWaveFrameType.BeamStart:
							style.backgroundColor = 'rgba(255, 255, 255, 0.1)'
							break
						case ZWaveFrameType.BeamStop:
							style.backgroundColor = 'rgba(0, 0, 0, 0.1)'
							break
						case ZWaveFrameType.Broadcast:
							style.backgroundColor = 'rgba(165, 0, 255, 0.1)'
							break
					}
				}
			}

			return { style }
		},
		getPayloadTags(payload, prev = []) {
			const tags = [
				...prev,
				payload.tags?.join(' '),
				...(payload.encapsulated || []).map((e) =>
					this.getPayloadTags(e, prev),
				),
			]
			return tags.filter((t) => !!t).join(' > ')
		},
		bindTopPaneObserver() {
			const onTopPaneResize = (e) => {
				this.topPaneHeight = e[0].contentRect.height
				this.perPage = Math.ceil(
					(this.topPaneHeight - this.offsetTop) / this.rowHeight,
				)
			}

			this.roTopPane = new ResizeObserver(onTopPaneResize)
			this.roTopPane.observe(this.$refs.topPane)
		},
		bindScroll() {
			if (this.scrollWrapper) return
			// find v-data-table__wrapper element inside #framesTable
			const el = this.$refs.framesTable?.$el

			if (el) {
				const wrapper = el.querySelector('.v-table__wrapper')
				wrapper.addEventListener('scroll', this.onScroll.bind(this), {
					passive: true,
				})
				this.scrollWrapper = wrapper
			}
		},
		enableAutoScroll() {
			this.autoScroll = true
			this.scrollBottom()
		},
		scrollBottom() {
			if (this.scrollWrapper && this.autoScroll) {
				this.scrollWrapper.scrollTo(
					0,
					this.scrollWrapper.scrollHeight + this.rowHeight,
				)
			}
		},
		async onScroll(e) {
			// debounce if scrolling fast
			if (this.timeoutScroll) {
				clearTimeout(this.timeoutScroll)
			}

			this.timeoutScroll = setTimeout(async () => {
				const prevStart = this.start

				// rows to show
				const { scrollTop } = e.target
				const rows = Math.ceil(scrollTop / this.rowHeight)
				// start index
				this.start =
					rows + this.perPage >= this.totalFrames &&
					this.perPage <= this.totalFrames
						? this.totalFrames - this.perPage
						: rows

				const direction = prevStart <= this.start ? 'down' : 'up'
				if (this.autoScroll && direction === 'up') {
					this.autoScroll = false
				}
			}, 50)
		},
		getTimestamp(timestamp) {
			// format timestamp HH:mm:ss.fff
			const date = new Date(timestamp)
			const ms = date.getMilliseconds()
			return `${date.toTimeString().split(' ')[0]}.${ms
				.toString()
				.padEnd(3, '0')}`
		},
		getRoute,
		getType,
		getRssi,
		getProtocolDataRate,
		async sendAction(data = {}, { hideInfo } = {}) {
			return new Promise((resolve) => {
				if (this.socket.connected) {
					if (!hideInfo)
						this.showSnackbar(`API ${data.apiName} called`, 'info')

					this.socket.emit(
						socketActions.zniffer,
						data,
						(response) => {
							if (!response.success) {
								this.showSnackbar(
									`Error while calling ${data.apiName}: ${response.message}`,
									'error',
								)
							}
							resolve(response)
						},
					)
				} else {
					resolve({
						success: false,
						message: 'Socket disconnected',
					})
					this.showSnackbar('Socket disconnected', 'error')
				}
			})
		},
		async getFrames() {
			const response = await this.sendAction(
				{
					apiName: 'getFrames',
				},
				{
					hideInfo: true,
				},
			)

			if (response.success) {
				this.frames = []
				response.result.forEach(this.addFrame)
			}
		},
		async setFrequency() {
			const response = await this.sendAction(
				{
					apiName: 'setFrequency',
					frequency: this.frequency,
				},
				{ hideInfo: true },
			)

			if (response.success) {
				this.frequencySuccess = true
			}
		},
		async setLRChannelConfig() {
			const response = await this.sendAction(
				{
					apiName: 'setLRChannelConfig',
					channelConfig: this.lrChannelConfig,
				},
				{ hideInfo: true },
			)

			if (response.success) {
				this.lrChannelConfigSuccess = true
			}
		},
		async startZniffer() {
			// If in loaded state, confirm discarding current trace
			if (this.viewState === 'loaded' && this.frames.length > 0) {
				const confirmed = await this.app.confirm(
					'Discard loaded trace',
					'Are you sure you want to discard the currently loaded trace?',
					'warning',
				)
				if (!confirmed) return
			}

			const response = await this.sendAction({
				apiName: 'start',
			})

			if (response.success) {
				this.viewState = 'recording'
				this.showSnackbar(`Zniffer started`, 'success')
			}
		},
		async stopZniffer() {
			const response = await this.sendAction({
				apiName: 'stop',
			})

			if (response.success) {
				this.viewState = 'stopped'
				this.showSnackbar(`Zniffer stopped`, 'success')
			}
		},
		async clearFrames() {
			const response = await this.sendAction({
				apiName: 'clear',
			})

			if (response.success) {
				this.showSnackbar(`Zniffer cleared`, 'success')
				this.frames = []
				this.framesFiltered = []
				// If currently in stopped or loaded state, go to initial state
				if (
					this.viewState === 'stopped' ||
					this.viewState === 'loaded'
				) {
					this.viewState = 'initial'
				}
			}
		},
		async createCapture() {
			const response = await this.sendAction({
				apiName: 'saveCaptureToFile',
			})

			if (response.success) {
				const result = response.result
				this.showSnackbar(
					`Capture "${result.name}" created in store`,
					'success',
				)
				// Trigger download of the Zniffer capture
				const fileName = result.name.replace('.zlf', '')
				this.app.exportConfiguration(result.data, fileName, 'zlf')
			}
		},
		async loadCapture() {
			try {
				// State-based confirmation logic
				if (this.frames.length > 0) {
					if (this.viewState === 'recording') {
						const confirmed = await this.app.confirm(
							'Stop recording',
							'Are you sure you want to stop recording and discard the recorded frames?',
							'warning',
						)
						if (!confirmed) return
						// Stop recording first
						await this.stopZniffer()
					} else if (this.viewState === 'stopped') {
						const confirmed = await this.app.confirm(
							'Discard recorded frames',
							'Are you sure you want to discard the recorded frames?',
							'warning',
						)
						if (!confirmed) return
					}
				}

				const { data: buffer } = await this.app.importFile('buffer')

				if (!buffer) return // User cancelled file selection

				const response = await this.sendAction(
					{
						apiName: 'loadCaptureFromBuffer',
						buffer: Array.from(new Uint8Array(buffer)),
					},
					{
						hideInfo: true,
					},
				)

				if (response.success) {
					// Refresh frames to show loaded data
					this.viewState = 'loaded'
					await this.getFrames()
				} else {
					throw new Error(
						response.message || 'Failed to load capture',
					)
				}
			} catch (error) {
				this.showSnackbar(
					`Error loading capture: ${error.message}`,
					'error',
				)
			}
		},
	},
}
</script>

<style scoped>
#framesTable :deep(td) {
	white-space: nowrap;
	/* display: -webkit-box;
	-webkit-line-clamp: 3;
	-webkit-box-orient: vertical; */
	overflow: hidden;
}

.pane::-webkit-scrollbar,
#framesTable :deep(.v-table__wrapper::-webkit-scrollbar) {
	height: 5px;
	width: 8px;
	background: rgba(0, 0, 0, 0.233);
	padding-right: 10;
}

.pane::-webkit-scrollbar-thumb,
#framesTable :deep(.v-table__wrapper::-webkit-scrollbar-thumb) {
	background: v-bind('$vuetify.theme.current.colors.primary');
	border-radius: 1ex;
	-webkit-border-radius: 1ex;
}

.single-line {
	white-space: nowrap;
	overflow: hidden;
	text-overflow: ellipsis;
}
</style>

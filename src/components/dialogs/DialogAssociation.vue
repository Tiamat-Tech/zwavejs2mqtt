<template>
	<v-dialog v-model="_value" max-width="500px" persistent>
		<v-card>
			<v-card-title>
				<span class="text-h5">New Association</span>
			</v-card-title>

			<v-card-text>
				<v-container grid-list-md>
					<v-form v-model="valid" ref="form" validate-on="lazy">
						<v-row>
							<v-col cols="12">
								<v-select
									label="Node Endpoint"
									hint="Used to filter available groups"
									v-model="group.endpoint"
									persistent-hint
									:items="endpoints"
								></v-select>
							</v-col>

							<v-col cols="12">
								<v-select
									label="Group"
									hint="Node/Endpoint Group association to Add/Remove"
									persistent-hint
									v-model="group.group"
									:items="endpointGroups"
									:rules="[required]"
									return-object
								>
									<template #selection="{ item }">
										{{ item.raw.title }}
									</template>
									<template #item="{ item, props }">
										<v-list-item
											v-bind="props"
											:title="item.raw.title"
											:subtitle="
												item.raw.endpoint >= 0
													? getEndpointLabel(
															node,
															item.raw.endpoint,
														)
													: 'No Endpoint'
											"
										>
										</v-list-item>
									</template>
								</v-select>
							</v-col>
							<v-col v-if="group.group" cols="12">
								<p class="text-subtitle-1 pa-0 ma-0">
									Max associations:
									<strong>{{ maxNodes }}</strong>
								</p>
								<p class="text-subtitle-1 pa-0 ma-0">
									Actual associations:
									<strong>{{ nodesInGroup }}</strong>
								</p>
							</v-col>

							<v-col class="pa-0" v-if="nodesInGroup < maxNodes">
								<v-col cols="12">
									<v-combobox
										label="Target Node"
										v-model="group.target"
										:items="filteredNodes"
										return-object
										:rules="[required]"
										hint="Node to add to the association group"
										persistent-hint
										item-title="_name"
									></v-combobox>
								</v-col>

								<v-col
									v-if="
										group.group && group.group.multiChannel
									"
									cols="12"
								>
									<v-select
										v-model.number="group.targetEndpoint"
										persistent-hint
										label="Target Endpoint"
										hint="Target node endpoint"
										:items="targetEndpoints"
									></v-select>
								</v-col>

								<v-col v-if="associationError" cols="12">
									<v-alert
										text
										density="compact"
										type="error"
									>
										{{ associationError }}
									</v-alert>
								</v-col>
							</v-col>
						</v-row>
					</v-form>
				</v-container>
			</v-card-text>

			<v-card-actions>
				<v-spacer></v-spacer>
				<v-btn
					color="blue-darken-1"
					variant="text"
					@click="$emit('close')"
					>Cancel</v-btn
				>
				<v-btn
					color="blue-darken-1"
					variant="text"
					:disabled="nodesInGroup >= maxNodes || !!associationError"
					@click="handleAdd"
					>ADD</v-btn
				>
			</v-card-actions>
		</v-card>
	</v-dialog>
</template>

<script>
import { Protocols } from '@zwave-js/core'
import { mapState } from 'pinia'
import useBaseStore from '../../stores/base.js'
import { getAssociationAddress } from '../../lib/utils'
import { AssociationCheckResult } from '@zwave-js/cc'
import { getEnumMemberName } from '@zwave-js/shared'
import InstancesMixin from '../../mixins/InstancesMixin.js'

export default {
	mixins: [InstancesMixin],
	props: {
		modelValue: Boolean,
		associations: Array,
		node: Object,
	},
	watch: {
		modelValue() {
			this.$refs.form && this.$refs.form.resetValidation()
			this.resetGroup()
			this.associationError = ''
		},
		group: {
			deep: true,
			async handler() {
				const result = await this.$refs.form?.validate()
				if (result?.valid) {
					this.allowedAssociation()
				}
			},
		},
	},
	computed: {
		...mapState(useBaseStore, ['controllerNode', 'nodes']),
		filteredNodes() {
			return this.node.protocol === Protocols.ZWaveLongRange
				? [this.controllerNode]
				: this.nodes
						.filter(
							(n) =>
								n.id !== this.node.id &&
								n.protocol !== Protocols.ZWaveLongRange,
						)
						.sort((a, b) => a._name.localeCompare(b._name))
		},
		endpoints() {
			return this.getEndpointItems(this.node)
		},
		targetEndpoints() {
			return this.getEndpointItems(this.group.target, true)
		},
		maxNodes() {
			return this.group.group ? this.group.group.maxNodes : -1
		},
		nodesInGroup() {
			return this.group.group
				? this.associations.filter(
						(a) =>
							a.groupId === this.group.group.value &&
							a.endpoint === this.group.endpoint,
					).length
				: 0
		},
		endpointGroups() {
			let groups = []
			try {
				groups = this.node.groups
				const endpoint = this.group.endpoint
				groups = groups.filter(
					(g) =>
						g.endpoint === endpoint ||
						(endpoint === null &&
							g.endpoint === 0 &&
							!g.multiChannel),
				)
				// eslint-disable-next-line no-empty
			} catch (error) {}

			return groups
		},
		_value: {
			get() {
				return this.modelValue
			},
			set(val) {
				this.$emit('update:modelValue', val)
			},
		},
	},
	data() {
		return {
			valid: true,
			group: {},
			associationError: '',
			defaultGroup: { endpoint: null },
			required: (v) => !!v || 'This field is required',
		}
	},
	methods: {
		getAssociationAddress,
		async handleAdd() {
			const result = await this.$refs.form.validate()
			if (result.valid) {
				this.$emit('add', this.group)
			}
		},
		async allowedAssociation() {
			const association = this.group
			const target = !isNaN(association.target)
				? parseInt(association.target)
				: association.target?.id

			if (isNaN(target)) {
				this.associationError = ''
				return
			}

			const group = association.group

			if (!group) {
				this.associationError = ''
				return
			}

			const toAdd = { nodeId: target }

			if (group.multiChannel && association.targetEndpoint >= 0) {
				toAdd.endpoint = association.targetEndpoint
			}

			const args = [
				this.getAssociationAddress({
					nodeId: this.node.id,
					endpoint: association.endpoint,
				}),
				group.value,
				toAdd,
			]

			const response = await this.app.apiRequest(
				'checkAssociation',
				args,
				{
					showInfo: false,
				},
			)

			if (response.success) {
				const checkResult = response.result

				if (checkResult === AssociationCheckResult.OK) {
					this.associationError = ''
				} else if (
					checkResult ===
					AssociationCheckResult.Forbidden_SecurityClassMismatch
				) {
					this.associationError = `Association not allowed: Node ${this.node.id} does not have the same security class as Node ${target}!`
				} else if (
					checkResult ===
					AssociationCheckResult.Forbidden_DestinationSecurityClassNotGranted
				) {
					this.associationError = `Association not allowed: Node ${this.node.id} was not granted the highest security class of Node ${target}!`
				} else if (
					checkResult ===
					AssociationCheckResult.Forbidden_NoSupportedCCs
				) {
					this.associationError = `Association not allowed: Node ${this.node.id} sends no commands in this group that Node ${target} supports!`
				} else {
					// This should not happen, but just in case
					this.associationError = `Association not allowed: ${getEnumMemberName(
						AssociationCheckResult,
						checkResult,
					)}`
				}
			}
		},
		resetGroup() {
			this.group = Object.assign({}, this.defaultGroup)
		},
		getEndpointLabel(node, endpoint) {
			if (node && node.endpoints) {
				const ep = node.endpoints.find((e) => e.index === endpoint)
				if (ep) {
					return ep.label
				}
			}
			return 'Endpoint ' + endpoint
		},
		getEndpointItems(node, noEndpoint = false) {
			const endpoints = []

			if (noEndpoint) {
				endpoints.unshift({ title: 'No Endpoint', value: null })
			}

			if (node && node.endpoints) {
				for (const endpoint of node.endpoints) {
					endpoints.push({
						title: endpoint.label,
						value: endpoint.index,
					})
				}
			}

			return endpoints
		},
	},
}
</script>

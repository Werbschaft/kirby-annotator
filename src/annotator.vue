<template>
	<div class="annotator" :data-theme="theme" :data-color="currentColor" @mouseup="stopDragging">
		<div class="annotator-background"></div>
		<div v-if="!src" class="annotator-placeholder">
			<div class="annotator-placeholder-ctn">
				Please select an image
			</div>
		</div>
		<div v-if="src" class="annotator-toolbar">
			<div class="annotator-toolbar-inner">
				<div v-for="button in buttons" :class="['tool', button, {'selected': currentTool == button}]" @click="setTool(button)">
					<component :is="'icon-' + button"></component>
				</div>
				<div class="color">
					<div class="color-selected blue" @click="toggleSelector">
						<div class="circle"></div>
						<svg viewBox="0 0 16 16">
					        <use xlink:href="#icon-angle-down" />
					    </svg>
					</div>
					<div v-if="showSelector" class="color-selector">
						<ul class="colors">
							<li v-for="color in colors" :class="['color-entry', color]" @click="setColor(color)">
								<div class="circle"></div>
							</li>
						</ul>
					</div>
				</div>
			</div>
			<div v-if="debug" class="annotator-toolbar-debug">
				<div class="coord x">{{ coords.x }}</div>
				<div class="coord y">{{ coords.y }}</div>
			</div>
		</div>
		<div v-if="src" class="annotator-ctn" @mousemove="updateCoords">
			<div class="image">
				<div class="image-ctn">
					<div ref="markers" class="markers" @mousemove="updateCoords" @mousedown="addMarker">
						<component v-for="(marker, index) in markers" :key="index" :index="index" :current="index == drag.index" :marker="marker" :rotate="rotate" v-bind:is="'marker-' + marker.type" @initDragResize="initDragResize"></component>
					</div>
					<img :src="src" alt="">
				</div>
			</div>
		</div>
	</div>
</template>

<script>
import iconPin from './assets/svg/icon-pin-rounded.vue'
import iconRect from './assets/svg/icon-rect-rounded.vue'
import iconCircle from './assets/svg/icon-circle-rounded.vue'

import markerPin from './components/markers/pin.vue'
import markerRect from './components/markers/rect.vue'
import markerCircle from './components/markers/circle.vue'

export default {
	components: {markerPin, markerRect, markerCircle, iconPin, iconRect, iconCircle},
	data() { 
		return {
			showSelector: false,
			currentTool: '',
			manualColor: '',
			storedColor: '',
			rotate: 0,
			src: '',
			coords: {
				x: 0,
				y: 0,
				xabs: 0,
				yabs: 0,
			},
			drag: {
				index: Number,
				type: String,
				isDragging: false,
				isResizing: false,
				handle: String,
				xref: Number,
				yref: Number,
				wref: Number,
				href: Number,
				maxRadius: Number,
			},
			markers: [],
		}
	},
	props: {
		buttons: Array,
		colors: Array,
		storage: Object,
		theme: String,
		parent: String,
    	name: String,
    	debug: Boolean,
	},
	computed: {
		currentColor: function() {
			return this.storedColor != '' ? this.storedColor : this.manualColor 
		}
	},
	created() {
		this.currentTool = this.buttons[0]
		this.manualColor = this.colors[0]
		this.$annotator.registerAnnotator(this)
	},
	destroyed() {
	    this.$annotator.unregisterAnnotator(this)
	},
	methods: {
		toggleSelector() {
			this.showSelector = !this.showSelector
		},
		setColor(color) {
			if(this.storage.color) {
	        	this.$mapsection.updateFields('color', this.storage.color, color)
	        }
	        else {
	        	this.manualColor = color
	        }
			this.showSelector = false
		},
		setTool(tool) {
			this.currentTool = tool
		},
		updateCoords(e) {
			let _markers = this.$refs.markers

			let _bounds = _markers.getBoundingClientRect()
			let _width  = _markers.clientWidth
			let _height = _markers.clientHeight 

  			let xabs = e.clientX - _bounds.left
  			let yabs = e.clientY - _bounds.top
  			let x    = xabs / _width
  			let y    = yabs / _height

  			x = Math.max(0, Math.min(1, x.toFixed(4)))
  			y = Math.max(0, Math.min(1, y.toFixed(4)))

			this.coords.x = x
			this.coords.y = y
			this.coords.xabs = xabs
			this.coords.yabs = yabs

			if(this.drag.isDragging) this.dragMarker(e)
			if(this.drag.isResizing) this.resizeMarker(e)
		},
		addMarker(e) {
			let _target = e.target
			if (e.target != this.$refs.markers) return false
			if (e.which != 1) {
				e.preventDefault()
				return false
			} 

			// create a new marker
			let newMarker = {
				type: this.currentTool,
				x: this.coords.x,
				y: this.coords.y,
				w: 0,
				h: 0,
			};
			this.markers.push(newMarker)

			// enable resizing
			this.drag.index        = this.markers.length - 1
			this.drag.isResizing   = true
			this.drag.type         = this.currentTool
			this.drag.handle       = 'bottom-right'
			this.drag.xref         = this.coords.x
			this.drag.yref         = this.coords.y
			this.drag.maxRadius    = this.getClosestSide(this.coords.xabs, this.coords.yabs)
		},
		getClosestSide(xabs, yabs) {
			let _markers = this.$refs.markers
			let _width  = _markers.clientWidth
			let _height = _markers.clientHeight

			let distanceTop = yabs
			let distanceBottom = _height - yabs
			let distanceLeft = xabs
			let distanceRight = _width - xabs

			return Math.min(distanceTop, distanceBottom, distanceLeft, distanceRight)
		},
		dragMarker(e) {
			let _marker = this.markers[this.drag.index]

			if (this.drag.type == 'pin') {
				_marker.x = this.coords.x
				_marker.y = this.coords.y
			}
			if (this.drag.type == 'circle') {
				let _markers = this.$refs.markers
				let minX = _marker.w / 2
				let maxX = 1 - minX
				let minY = _marker.h / 2
				let maxY = 1 - minY

				let x = Math.min(maxX, Math.max(minX, this.coords.x))
				let y = Math.min(maxY, Math.max(minY, this.coords.y))
				_marker.x = x
				_marker.y = y
			}
		},
		resizeMarker(e) {
			let _marker = this.markers[this.drag.index]

			if (this.drag.type == 'rect') {
				let x = parseFloat(_marker.x)
				let y = parseFloat(_marker.y)
				let w = parseFloat(_marker.w)
				let h = parseFloat(_marker.h)
				let xref = parseFloat(this.drag.xref)
				let yref = parseFloat(this.drag.yref)
				let wref = parseFloat(this.drag.wref)
				let href = parseFloat(this.drag.href)

				if (this.drag.handle == 'bottom-right') {
					w = this.coords.x - xref
					h = this.coords.y - yref
				}
				else if (this.drag.handle == 'top-left') {
					x = Math.min(xref + wref, this.coords.x)
					y = Math.min(yref + href, this.coords.y)

					w = wref + (xref - x)
					h = href + (yref - y)
				}
				else if (this.drag.handle == 'top-right') {
					y = Math.min(yref + href, this.coords.y)

					w = wref + (this.coords.x - (xref + wref))
					h = href + (yref - y)
				}
				else if (this.drag.handle == 'bottom-left') {
					x = Math.min(xref + wref, this.coords.x)

					w = wref + (xref - x)
					h = this.coords.y - yref
				}

				_marker.x = Math.max(0, Math.min(1, x.toFixed(4)))
				_marker.y = Math.max(0, Math.min(1, y.toFixed(4)))
				_marker.w = Math.max(0, Math.min(1, w.toFixed(4)))
				_marker.h = Math.max(0, Math.min(1, h.toFixed(4)))
			}
			else if (this.drag.type == 'circle') {
				let _markers = this.$refs.markers
				let _width  = _markers.clientWidth
				let _height = _markers.clientHeight

				// pythagore with absolute values
				let w = this.coords.xabs - (this.drag.xref * _width)
				let h = this.coords.yabs - (this.drag.yref * _height)
				let r = Math.sqrt(Math.pow(w, 2) + Math.pow(h, 2))

				// limit to max radius
				r = Math.min(this.drag.maxRadius, r)
				// back to percent
				r = r / _width

				// width is radius * 2
				w = r * 2
				// height is width * ratio
				h = w * _width / _height

				_marker.w = w.toFixed(4)
				_marker.h = h.toFixed(4)

				// set an angle
				let angleDeg = Math.atan2(this.coords.yabs - (this.drag.yref * _height), this.coords.xabs - (this.drag.xref * _width)) * 180 / Math.PI
				this.rotate = angleDeg.toFixed(4)
			}			
		},
		initDragResize(val) {
			console.log(val)
			// enable resizing
			this.drag.index        = val.index
			this.drag.isDragging   = val.drag
			this.drag.isResizing   = val.resize
			this.drag.type         = val.type

			if(val.resize) {
				let _markers = this.$refs.markers
				let _marker  = this.markers[this.drag.index]

				if(val.type == 'circle') {
					this.drag.xref         = _marker.x
					this.drag.yref         = _marker.y
					this.drag.maxRadius    = this.getClosestSide(_marker.x * _markers.clientWidth, _marker.y * _markers.clientHeight)
				}
				if(val.type == 'rect') {
					this.drag.handle       = val.handle
					this.drag.xref         = _marker.x
					this.drag.yref         = _marker.y
					this.drag.wref         = _marker.w
					this.drag.href         = _marker.h
				}
			}			
		},
		stopDragging(e) {
			if (!this.drag.isDragging && !this.drag.isResizing) return false

			this.updateStructure()

			this.drag.index        = Number
			this.drag.isDragging   = false
			this.drag.isResizing   = false
			this.drag.type         = String
			this.drag.handle       = String
			//this.drag.xref         = Number
			//this.drag.yref         = Number
			this.drag.maxRadius    = Number

			this.rotate = 0
		},
		setValue(fieldname, value) {
	        try {
		        for (let datapoint in this.storage) {
		            if (this.storage[datapoint] === fieldname) {
			            switch(datapoint) {
			                case 'src':
			                	this.src = value
			                	break
			                case 'color':
			                	this.storedColor = value
			                	break
			                case 'markers':
			                	if(!Array.isArray(value)) {
			                    	console.warn('could not adapt field to markers datapoint, not an array: ', value)
			                    	break
			                    }
				                this.markers = value.map(entries => ({
				                    type: entries.type,
									x: entries.x,
									y: entries.y,
									w: entries.w,
									h: entries.h,
				                }))
			                	break
			                default:
			                	this[datapoint] = value
			                    break
			            }
		            }
		        }
	        } 
	        catch(e) {
	        	console.warn(e)
	        }
	    },
	    updateStructure() {
	        if(this.storage.markers) {
	        	this.$mapsection.updateFields('markers', this.storage.markers, this.markers)
	        }
	    },
	},
}
</script>

<style lang="scss">
	@import './assets/css/styles.scss'
</style>
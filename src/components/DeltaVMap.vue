<template>
  <div id="page-container">
    <!--    <v-container>-->
    <!--      <h2 class="text-uppercase my-8">delta v map</h2>-->
    <!--      <div class="intro text&#45;&#45;left mx-auto">-->
    <!--        <p>The following can be used to roughly calculate the-->
    <!--          <a href="https://en.wikipedia.org/wiki/Delta-v">delta-v</a>-->
    <!--          (change of velocity) requirements,-->
    <!--        to move from one orbit to another.-->
    <!--          </p>-->
    <!--        <p>-->
    <!--          <a href="">Hohmann transfer orbits</a> are used, which are-->
    <!--          normally the most efficient way to get from A to B.</p>-->
    <!--        <p>First click on the place of origin orbit and the destination orbit-->
    <!--        to see the total delta-v requirement.</p>-->
    <!--      </div>-->
    <!--    </v-container>-->
    <h1 class="site-title">delta v map</h1>
    <div class="controls text-left">
      <div class="controls__section controls__section--origin" v-show="$vuetify.breakpoint.mdAndUp || !selectedA">
        <label class="controls__label">origin</label>
        <div class="controls__value "
             :class="{'controls__value--active': selectedA}">
          <span v-show="!selectedAText"
                class="controls__placeholder controls__placeholder--origin">Select origin</span>
          &#8203;{{ selectedAText }}
        </div>
      </div>
      <div class="controls__section controls__section--destination" v-show="$vuetify.breakpoint.mdAndUp || (selectedA && !selectedB)">
        <label class="controls__label">destination</label>
        <div class="controls__value "
             :class="{'controls__value--active': selectedB}">
          <span v-show="!selectedBText"
                class="controls__placeholder controls__placeholder--destination">Select destination</span>
          &#8203;{{ selectedBText }}
        </div>
      </div>
      <div class="controls__section" v-show="$vuetify.breakpoint.mdAndUp || (selectedA && selectedB)">
        <label class="controls__label">delta v</label>
        <div class="controls__value" :class="{'controls__value--active': deltaV}" >&#8203;{{ deltaVText }}</div>
      </div>
      <div class="controls__section controls__aerobraking"
           :class="{'controls__aerobraking--active': aeroBrakingAvailable}"
           v-show="$vuetify.breakpoint.mdAndUp || (selectedA && selectedB)"
      >
        <div class="controls__value">
          <span class="controls__aerobraking_check">✓</span><span class="controls__aerobraking_label">Aero-breaking available</span>
        </div>
      </div>
      <div class="controls__section controls__buttons d-flex justify-space-between">
        <div><v-btn small
                    color="grey lighten-1"
                    :disabled="!selectedA || !selectedB"
                    @click="reverseSelectedNodes">reverse</v-btn></div>
        <div><v-btn small
                    color="grey lighten-1"
                    :disabled="!selectedA && !selectedB"
                    @click="clearSelectedNodes">clear</v-btn></div>
      </div>
    </div>
    <div class="map-container" :class="{'map-container--visible': cytoscapeLoaded}">
<!--      <div class="map-container__prompt map-container__prompt&#45;&#45;origin" v-show="!selectedA && !selectedB">Select origin</div>-->
<!--      <div class="map-container__prompt map-container__prompt&#45;&#45;destination" v-show="selectedA && !selectedB">Select destination</div>-->
      <div id="map" class="map" :class="{'path-selected' : pathSelected}"></div>
    </div>
  </div>
</template>
<script>
// https://learnvue.co/2020/09/how-to-deploy-your-vue-app-to-github-pages/
// https://www.reddit.com/r/space/comments/1ktjfi/deltav_map_of_the_solar_system/
// https://dash.plotly.com/cytoscape
// https://imgur.com/a/ordID
// https://launchercalculator.com/

// delta v calculators
// https://space.geometrian.com/calcs/opt-multi-stage.php
// https://www.desmos.com/calculator/hib7psndtb

import cytoscape from 'cytoscape'
import cytoscapeFcose from 'cytoscape-fcose'

import OrbitsArray from './nodes'
import DeltasArray from './edges'
import CreateOuterPlanets from './create-outer-planets'
import CreateCytoscapeStyles from './create-cytoscape-styles'

cytoscape.use(cytoscapeFcose)

const colors = {
  pathColor: 'mediumpurple', // 'lightskyblue' // 'lightgreen'
  originColor: 'lightskyblue',
  destinationColor: 'orange'
}

export default {
  data () {
    return {
      svg: null,
      width: 500,
      selectedA: null,
      selectedB: null,
      deltaV: null,
      fixedNodeConstraints: [],
      orbits: [],
      deltas: [],
      jupiterSystem: [],
      saturnSystem: [],
      uranusSystem: [],
      colDelta: 250, // layouts
      planetY: 0,
      planetYDelta: 350,
      pathSelected: false,
      aeroBrakingAvailable: false,
      opacityLevel: 0.1,
      cytoscapeLoaded: false
    }
  },
  computed: {
    formattedData: function () {
      const systemData = this.formatSystems(this.systems)
      const orbitData = this.formatOrbits(this.orbits)
      const deltaData = this.formatDeltas(this.deltas)
      const allData = [
        ...systemData,
        ...orbitData,
        ...deltaData,
        ...this.jupiterSystem.nodesAndEdges,
        ...this.saturnSystem.nodesAndEdges,
        ...this.uranusSystem.nodesAndEdges,
        ...this.neptuneSystem.nodesAndEdges,
        ...this.plutoSystem.nodesAndEdges,
        ...this.haumeaSystem.nodesAndEdges,
        ...this.makemakeSystem.nodesAndEdges
      ]
      return allData
    },
    selectedAText: function () { return (this.selectedA) ? this.selectedA.label : '' },
    selectedBText: function () { return (this.selectedB) ? this.selectedB.label : '' },
    deltaVText: function () { return (this.deltaV) ? this.deltaV.toFixed(2) + ' km/s' : '' }
  },
  methods: {
    setPlanetY: function (amount) {
      this.planetY = amount
      return this.planetY
    },
    incPlanetYDelta: function (amount = this.planetYDelta) {
      this.setPlanetY(this.planetY + amount); return this.planetY
    },
    createFixedNodeConstraints: function () {
      const col = n => n * this.colDelta
      const planetX = col(1)
      const incPlanetY = this.incPlanetYDelta
      const lOrbitX = col(2)
      const captureX = col(3)
      const transferX = col(6)
      const moonTransferX = col(4)

      const constraints = [] // position constrains
      const c = (nodeId, x, y) => { constraints.push({ nodeId, position: { x, y } }) }
      c('Sun', planetX, this.planetY)
      c('LSunO', lOrbitX, this.planetY)
      c('SunT', transferX, this.planetY)

      c('Merc', planetX, incPlanetY())
      c('LMercO', lOrbitX, this.planetY)
      c('MercCE', captureX, this.planetY)
      c('MercT', transferX, this.planetY)

      c('Venus', planetX, incPlanetY())
      c('LVenusO', lOrbitX, this.planetY)
      c('VenusCE', captureX, this.planetY)
      c('VenusT', transferX, this.planetY)

      const earthY = incPlanetY()
      const moonY = incPlanetY(2 * this.planetYDelta)
      c('Earth', planetX, earthY)
      c('LEO', lOrbitX, earthY)
      c('EarthCE', transferX, moonY)
      c('GTO', col(4), earthY)
      c('GEO', col(5), earthY)

      c('MoonT', col(4), moonY)
      c('EarthMoonL1', col(3), earthY + this.planetYDelta)
      c('EarthMoonL2', col(3), moonY + this.planetYDelta)
      c('Moon', planetX, moonY)
      c('LMoonO', lOrbitX, moonY)
      c('MoonCE', col(3), this.planetY)

      c('Deimos', planetX, incPlanetY(2 * this.planetYDelta))
      c('MarsCE', col(5), this.planetY)
      c('MarsT', transferX, this.planetY)
      c('LDeimosO', lOrbitX, this.planetY)
      c('DeimosCE', captureX, this.planetY)
      c('DeimosT', moonTransferX, this.planetY)

      c('Phobos', planetX, incPlanetY())
      c('LPhobosO', lOrbitX, this.planetY)
      c('PhobosCE', captureX, this.planetY)
      c('PhobosT', col(4), this.planetY)

      c('Mars', planetX, incPlanetY())
      c('LMarsO', lOrbitX, this.planetY)

      const vestaY = incPlanetY()
      c('VestaT', col(6), vestaY)
      c('VestaCE', col(3), vestaY)
      c('LVestaO', col(2), vestaY)
      c('Vesta', col(1), vestaY)

      const ceresY = incPlanetY()
      c('CeresT', col(6), ceresY)
      c('CeresCE', col(3), ceresY)
      c('LCeresO', col(2), ceresY)
      c('Ceres', col(1), ceresY)

      this.planetX = col(8)
      return constraints
    },
    getFixedNodeConstraints: function () { return this.fixedNodeConstraints },
    addFixedNodeConstraint: function (c) { this.fixedNodeConstraints.push(c) },
    addFixedNodeConstraints: function (a) { a.map(c => this.addFixedNodeConstraint(c)) },
    createSystem: function (id, label, parent = null) {
      return {
        id,
        label,
        parent,
        selected: false,
        nodeType: 'system'
      }
    },
    createFurnishedSystem: function (id, label, parent = null) {
      return this.furnishSystemObject(
        this.formatData(this.createSystem(id, label, parent))
      )
    },
    createOuterPlanetMoonSystem: function (
      name, hasAtmosphere,
      predecessorTransferOrbitId, predecessorHasAtmosphere,
      outerPlanetTransferDV, outerPlanetCaptureDV,
      color, moonsArray,
      lowOrbitDV, lowOrbitAltitude,
      surfaceDV,
      layoutStartX,
      layoutStartY,
      yOffsetInt
    ) {
      const positionConstraints = []
      const createConstraint = (nodeId, x, y) => {
        positionConstraints.push({ nodeId, position: { x, y } })
      }
      const col = n => layoutStartX + (n * this.colDelta)
      let currentCol = 0

      // create the outer planet system and the giant
      const thisSys = name + 'Sys'
      const outerPlanetSystem = this.createFurnishedSystem(
        thisSys, name + ' System', 'SunSys'
      )

      // create outer planet
      const planetSurface = this.furnishedOrbitObject({
        id: name, label: name, nodeType: 'surface', parent: thisSys, color
      })

      // create the outer planet transfer orbit and delta
      const transferName = name + 'T'
      const outerPlanetTransfer = this.furnishedOrbitObject(
        { id: transferName, label: name + ' Transfer', nodeType: 'orbit-transfer', parent: 'SunSys' }
      )
      createConstraint(transferName, col(0), layoutStartY)
      let predecessorTransferAB = 0
      if (predecessorHasAtmosphere) {
        if (hasAtmosphere) {
          predecessorTransferAB = 2
        } else {
          predecessorTransferAB = 1
        }
      } else {
        if (hasAtmosphere) {
          predecessorTransferAB = 3
        } else {
          predecessorTransferAB = 0
        }
      }
      const planetAB = (hasAtmosphere) ? 1 : 0
      const outerPlanetTransferDeltaObject = this.furnishedDeltaObject(
        transferName, predecessorTransferOrbitId, outerPlanetTransferDV, predecessorTransferAB
      )

      // create outer planet capture orbit and delta
      const captureName = name + 'CE'
      const outerPlanetCaptureEscape = this.furnishedOrbitObject(
        {
          id: captureName,
          label: name + ' Capture/Escape',
          nodeType: 'orbit-capture-escape',
          parent: thisSys
        }
      )
      currentCol = ++currentCol + yOffsetInt
      createConstraint(captureName, col(currentCol++), layoutStartY)
      const outerPlanetCaptureDeltaObject = this.furnishedDeltaObject(
        transferName, captureName, outerPlanetCaptureDV, planetAB
      )

      const moonTransferY = layoutStartY
      const moonCaptureY = moonTransferY + this.planetYDelta
      const moonLowOrbitY = moonCaptureY + this.planetYDelta
      const moonSurfaceY = moonLowOrbitY + this.planetYDelta

      let prevSource = outerPlanetCaptureEscape
      let transferDelta = outerPlanetCaptureDV
      const moons = moonsArray.map(m => {
        const moonName = m[1]
        const ab = (typeof m[6] !== 'undefined') ? m[6] : false // moon aeroBraking availability
        const moonHighTransfer = this.furnishedOrbitObject({ id: moonName + 'T', label: moonName + ' Transfer', nodeType: 'orbit-transfer', parent: thisSys })
        const moonHighTransferDelta = this.furnishedDeltaObject(prevSource.data.id, moonHighTransfer.data.id, transferDelta, planetAB)
        createConstraint(moonHighTransfer.data.id, col(currentCol), moonTransferY)
        const moonLowTransfer = this.furnishedOrbitObject({ id: moonName + 'T', label: moonName + ' Transfer', nodeType: 'orbit-transfer', parent: thisSys })
        const moonCapture = this.furnishedOrbitObject({ id: moonName + 'CE', label: moonName + ' Capture/Escape', nodeType: 'orbit-capture-escape', parent: thisSys })
        const moonCaptureDelta = this.furnishedDeltaObject(moonLowTransfer.data.id, moonCapture.data.id, m[2], ab)
        createConstraint(moonCapture.data.id, col(currentCol), moonCaptureY)
        const moonLowOrbit = this.furnishedOrbitObject({ id: 'L' + moonName + 'O', label: moonName + ' Low Orbit', nodeType: 'orbit', parent: thisSys, altitude: m[4] })
        createConstraint(moonLowOrbit.data.id, col(currentCol), moonLowOrbitY)
        const moonLowDelta = this.furnishedDeltaObject(moonCapture.data.id, moonLowOrbit.data.id, m[3], ab)
        const moonSurface = this.furnishedOrbitObject({ id: moonName, label: moonName, nodeType: 'surface', parent: thisSys, color: '#807E7F' })
        createConstraint(moonSurface.data.id, col(currentCol), moonSurfaceY)
        const moonSurfaceDelta = this.furnishedDeltaObject(moonLowOrbit.data.id, moonSurface.data.id, m[5], ab)
        prevSource = moonHighTransfer
        transferDelta = m[0]
        currentCol++
        return [
          moonHighTransfer,
          moonHighTransferDelta,
          moonLowTransfer,
          moonCaptureDelta,
          moonCapture,
          moonLowDelta,
          moonLowOrbit,
          moonSurface,
          moonSurfaceDelta
        ]
      })

      const lowOrbitName = 'L' + name + 'O'
      createConstraint(lowOrbitName, col(currentCol), layoutStartY)
      createConstraint(name, col(++currentCol), layoutStartY)
      const nodesAndEdges = [
        outerPlanetSystem,
        planetSurface,
        outerPlanetTransfer,
        outerPlanetTransferDeltaObject,
        outerPlanetCaptureEscape,
        outerPlanetCaptureDeltaObject,
        ...[].concat(...moons),
        this.furnishedOrbitObject({
          id: lowOrbitName, label: name + ' Low Orbit', nodeType: 'orbit', parent: thisSys, altitude: lowOrbitAltitude
        }),
        this.furnishedDeltaObject(prevSource.data.id, lowOrbitName, lowOrbitDV, planetAB),
        this.furnishedDeltaObject(lowOrbitName, name, surfaceDV, planetAB)
      ]

      return {
        nodesAndEdges,
        positionConstraints
      }
    },
    formatData: function (o) {
      return { data: o }
    },
    formatDataArray: function (arrayData) {
      return arrayData.map(o => {
        return this.formatData(o)
      })
    },
    furnishSystemObject: function (formatedSystemObject) {
      const o = formatedSystemObject
      o.events = false
      o.classes = 'top-center'
      o.grabbable = false
      o.selectable = false
      o.pannable = true
      return o
    },
    furnishedSystemObject: function (unformatedSystemObject) {
      return this.furnishSystemObject(this.formatData(unformatedSystemObject))
    },
    formatSystems: function (unformatedSystemsArray) {
      return unformatedSystemsArray.map(o => {
        return this.furnishedSystemObject(o)
      })
    },
    furnishOrbitObject: function (formatedObject) {
      const o = formatedObject
      o.classes = 'top-center'
      o.grabbable = false
      o.selectable = false
      if (typeof o.data.altitude !== 'undefined') {
        o.data.label = o.data.label + ' (' + o.data.altitude + 'km)'
      }
      return o
    },
    furnishedOrbitObject: function (unformatedOrbitObject) {
      return this.furnishOrbitObject(this.formatData(unformatedOrbitObject))
    },
    formatOrbits: function (unformatedOrbitArray) {
      return unformatedOrbitArray.map(o => {
        return this.furnishedOrbitObject(o)
      })
    },
    createDeltaObject: function (source, target, dv, ab = 0) {
      switch (ab) {
        case 0: ab = 'false'; break
        case 1: ab = 'true'; break
        case 2: ab = 'both'; break
        case 3: ab = 'reverse'; break
      }
      return { source, target, dv, ab }
    },
    createDeltaDataObject: function (source, target, dv, ab = 0) {
      return { data: this.createDeltaObject(source, target, dv, ab) }
    },
    furnishDeltaObject: function (o) {
      o.data.label = o.data.dv
      o.selectable = false
      return o
    },
    furnishedDeltaObject: function (source, target, dv, ab = 0) {
      return this.furnishDeltaObject(
        this.createDeltaDataObject(source, target, dv, ab)
      )
    },
    formatDeltas: function (deltaArrays) {
      // convert array to objects
      const newDeltaArray = deltaArrays.map(d => {
        return this.createDeltaObject(d[0], d[1], d[2], d[3])
      })
      return this.formatDataArray(newDeltaArray).map(o => {
        return this.furnishDeltaObject(o)
      })
    },
    createData: function () {
      this.systems = [
        this.createSystem('SunSys', 'Sun System'),
        this.createSystem('MercurySys', 'Mercury System', 'SunSys'),
        this.createSystem('VenusSys', 'Venus System', 'SunSys'),
        this.createSystem('EarthSys', 'Earth System', 'SunSys'),
        this.createSystem('MoonSys', 'Moon System', 'EarthSys'),
        this.createSystem('DeimosSys', 'Deimos System', 'MarsSys'),
        this.createSystem('MarsSys', 'Mars System', 'SunSys'),
        this.createSystem('PhobosSys', 'Phobos System', 'MarsSys'),
        this.createSystem('VestaSys', 'Vesta System', 'SunSys'),
        this.createSystem('CeresSys', 'Ceres System', 'SunSys')
      ]
      this.orbits = OrbitsArray
      this.deltas = DeltasArray
      this.fixedNodeConstraints = this.createFixedNodeConstraints()
      CreateOuterPlanets(this)
    },
    handleBothTerminalsAlreadySelected: function (nodeData) {
      console.log('handleBothTerminalsAlreadySelected')
      this.clearSelectedNodes()
      this.selectedA = nodeData
      this.selectedB = null
      this.deltaV = null
      const node = this.cy.$('#' + nodeData.id)
      node.addClass('node-on-path')
      node.addClass('origin-node')
    },
    handleReceivedOriginTerminal: function (nodeData) {
      console.log('handleReceivedOriginTerminal')
      this.selectedA = nodeData
      const node = this.cy.$('#' + nodeData.id)
      node.addClass('node-on-path')
      node.addClass('origin-node')
    },
    testIfAeroBrakingIsAvailable: function (edge, previousNode) {
      if (edge.ab === 'both') {
        this.aeroBrakingAvailable = true
      } else {
        // delta is going with the path direction
        if (previousNode.id === edge.source && edge.ab === 'true') {
          this.aeroBrakingAvailable = true
        }

        // delta is going against the path direction
        if (previousNode.id === edge.target && edge.ab === 'reverse') {
          this.aeroBrakingAvailable = true
        }
      }
    },
    handleReceivedDestinationTerminal: function (nodeData) {
      console.log('handleReceivedDestinationTerminal')
      const self = this
      self.aeroBrakingAvailable = false
      self.pathSelected = true
      self.selectedB = nodeData
      const dijkstra = self.cy.elements().dijkstra(
        `#${self.selectedA.id}`,
        function (el) {
          return parseFloat(el._private.data.dv)
        }, false
      )
      self.cy.$('#' + self.selectedB.id).addClass('destination-node')
      self.cy.batch(function () {
        self.cy.$('#SunSys').addClass('path-selected')
      })

      const pathToB = dijkstra.pathTo(`#${self.selectedB.id}`)
      let previousNode = null
      pathToB.map(el => {
        const obj = el._private.data
        const id = obj.id
        const cyObj = self.cy.$('#' + id)

        // handle node
        if (el._private.data.nodeType) {
          cyObj.addClass('node-on-path')
          previousNode = obj

          // handle edge
        } else {
          cyObj.addClass('edge-on-path')

          // don't check again if already found to be true once
          if (self.aeroBrakingAvailable === false) {
            const edge = el._private.data
            self.testIfAeroBrakingIsAvailable(edge, previousNode)
          }
        }
      })
      // let offset = 0
      // const getOffset = function () {
      //   offset++
      //   return offset
      // }
      // self.interval = setInterval(function () {
      //   self.cy.batch(function () {
      //     self.cy.$('.edge-on-path').style({
      //       'line-dash-offset': getOffset
      //       // duration: 100000,
      //       // step: function (thing) {
      //       //   console.log('anim step', thing, offset)
      //       //   offset++
      //       // }
      //     })
      //   })
      // }, 10)
      self.cy.batch(function () {
        const allEdges = self.cy.$('edge')
        console.log('all edges length', allEdges.length)
        const edgesOnPath = self.cy.$('.edge-on-path')
        console.log('edges on path length', edgesOnPath.length)
        const edgesNotOnPath = allEdges.difference(edgesOnPath)
        console.log('edges not on path length', edgesNotOnPath.length)
        edgesNotOnPath.css('opacity', self.opacityLevel)
      })
      const distToB = dijkstra.distanceTo(`#${self.selectedB.id}`)
      self.deltaV = distToB
    },
    nodeSelected: function (target) {
      console.log('nodeSelected')
      const nodeData = target._private.data

      if (nodeData.nodeType === 'system') {
        this.clearSelectedNodes()
        return
      }

      // restarting again
      if (this.selectedA && this.selectedB) {
        this.handleBothTerminalsAlreadySelected(nodeData)
      } else {
        // already have the origin
        if (this.selectedA) {
          this.handleReceivedDestinationTerminal(nodeData)
        } else {
          this.handleReceivedOriginTerminal(nodeData)
        }
      }
    },
    findById: function (id) {
      return this.cy.$('#' + id)
    },
    findByClass: function (className) {
      return this.cy.$('.' + className)
    },
    findAndRemoveClass: function (className) {
      this.findByClass(className).removeClass(className)
    },
    reverseSelectedNodes: function () {
      const originalA = this.selectedA
      const originalB = this.selectedB
      this.findAndRemoveClass('origin-node')
      this.findAndRemoveClass('destination-node')
      // this.clearSelectedNodes()
      this.selectedA = originalB
      this.selectedB = originalA
      this.findById(this.selectedA.id).addClass('origin-node')
      this.findById(this.selectedB.id).addClass('destination-node')
      this.handleReceivedDestinationTerminal(this.selectedB)
    },
    clearSelectedNodes: function () {
      this.pathSelected = false
      this.selectedA = null
      this.selectedB = null
      this.deltaV = null
      this.aeroBrakingAvailable = false
      clearInterval(this.interval)
      const self = this
      this.cy.batch(function () {
        self.cy.$(':selected').unselect()
        self.findAndRemoveClass('origin-node')
        self.findAndRemoveClass('destination-node')
        self.findAndRemoveClass('node-on-path')
        self.findAndRemoveClass('edge-on-path')
        self.findAndRemoveClass('path-selected')

        self.cy.$('edge').css('opacity', 1)
      })
    }
  },
  mounted () {
    const self = this
    self.createData()
    const map = document.getElementById('map')
    const cy = cytoscape({
      container: map, // container to render in
      elements: this.formattedData,
      wheelSensitivity: 0.25, // TODO make this user configurable
      style: CreateCytoscapeStyles(
        this, colors),
      layout: {
        name: 'fcose',
        quality: 'proof',
        edgeElasticity: 0.1,
        nodeRepulsion: 40000,
        nodeSeparation: 200,
        fixedNodeConstraint: this.getFixedNodeConstraints()
      }
    })
    window.cy = cy
    self.cy = cy
    cy.on('mouseover', 'node', function (e) { map.style.cursor = 'pointer' })
    cy.on('mouseout', 'node', function (e) { map.style.cursor = 'default' })
    cy.on('tap', "node[type != 'system']", (e) => { self.nodeSelected(e.target) })
    setTimeout(() => { this.cytoscapeLoaded = true }, 1000)
  }
}
</script>
<style lang="sass" scoped>
@import '~vuetify/src/styles/styles.sass'

$padding: 1rem
$color-offwhite: #eee
$color-light: #ccc
$color-dark: #333
$controls-width: 300px
$color-origin: lightskyblue
$color-destination: orange

.light-box-shadow
  box-shadow: 0px 1px 20px 0px #666

*
  box-sizing: border-box
  font-family: "Roboto", sans-serif
  transition: background-color .25s, color .25s, opacity .25s

#page-container
  background-color: $color-light
  display: grid
  height: 100vh
  width: 100vw

  &:before
    @extend .light-box-shadow
    bottom: 0
    content: ''
    left: 0
    position: absolute
    right: 0
    top: 0
    width: $controls-width
    z-index: 2

  @media #{map-get($display-breakpoints, 'sm-and-down')}
    grid-auto-rows: 50px minmax(0, 1fr) auto
    &:before
      display: none

  @media #{map-get($display-breakpoints, 'md-and-up')}
    grid-auto-rows: auto minmax(0, 1fr)
    grid-template-columns: $controls-width auto

.site-title
  @extend .light-box-shadow
  background-color: $color-light
  text-transform: uppercase
  z-index: 3
  @media #{map-get($display-breakpoints, 'sm-and-down')}
    font-size: 1.25rem
    line-height: 2.25rem
    padding: .5rem 1rem
    text-align: left

  @media #{map-get($display-breakpoints, 'md-and-up')}
    box-shadow: none
    font-size: 1.5rem
    grid-col-start: 1
    grid-col-end: 1
    padding: 1rem 0
    text-align: center
    width: $controls-width

.controls
  @extend .light-box-shadow
  $shadow: 2px
  background-color: $color-light
  color: $color-dark
  display: flex
  z-index: 2
  @media #{map-get($display-breakpoints, 'sm-and-down')}
    display: flex
    flex-wrap: wrap
    grid-row: 3
    justify-content: center
    padding: .5rem 0
    width: 100%
  @media #{map-get($display-breakpoints, 'md-and-up')}
    box-shadow: none
    display: flex
    flex-direction: column
    grid-col: 1
    grid-row: 2
    padding: 0 1rem
    width: $controls-width

  .row
    padding: .5rem

  .controls
    &__section
      @media #{map-get($display-breakpoints, 'sm-and-down')}
        align-items: center
        display: flex
        margin: .5rem 1rem

      @media #{map-get($display-breakpoints, 'md-and-up')}
        margin-top: .5rem

      &--origin, &--destination
        .controls__value
          &--active
            overflow: hidden
            position: relative
            &:before
              content: ''
              position: absolute
              display: inline-block
              width: 5px
              left: 0
              top: 0
              bottom: 0
              background-color: $color-dark
          @media #{map-get($display-breakpoints, 'sm-and-down')}
            background-color: $color-dark
            border-radius: .5rem
            color: $color-light
            padding-left: .75rem
            position: relative
            &:before
              content: ''
              position: absolute
              display: inline-block
              width: .25rem
              left: 0
              top: 0
              bottom: 0

      &--origin
        .controls__value
          &--active
            &:before
              background-color: $color-origin
          @media #{map-get($display-breakpoints, 'sm-and-down')}
            &:before
              background-color: $color-origin

      &--destination
        .controls__value
          &--active
            &:before
              background-color: $color-destination
          @media #{map-get($display-breakpoints, 'sm-and-down')}
            &:before
              background-color: $color-destination

    &__label
      text-transform: capitalize
      @media #{map-get($display-breakpoints, 'sm-and-down')}
        display: none
        margin-right: 1rem
      @media #{map-get($display-breakpoints, 'md-and-up')}
        min-width: 6rem

    &__value
      border: 1px solid #bbb
      border-radius: .25rem
      min-width: 8rem
      overflow: hidden
      padding: .25rem .5rem
      text-align: center
      &--active
        background-color: $color-dark
        color: $color-light

    &__placeholder
      opacity: 0.7
      @media #{map-get($display-breakpoints, 'md-and-up')}
        display: none

    &__aerobraking
      opacity: 0.5
      text-align: left

      .controls__value
        border: 0
        padding-left: 0
        text-align: left

      &_check
        color: transparent
        display: inline-block
        margin-right: .5em
        border: 2px solid $color-dark
        border-radius: 2px
        width: 20px
        height: 20px
        line-height: 17px
        padding-left: 2px

      &--active
        opacity: 1
        color: $color-dark
        .controls__aerobraking_check
          color: $color-dark

    &__buttons
      > *
        @media #{map-get($display-breakpoints, 'sm-and-down')}
          margin-right: 1rem

.intro
  max-width: 500px
  p
    text-align: left

.map-container
  background-color: #888
  opacity: 0
  position: relative
  z-index: 1

  &--visible
    opacity: 1

  &__prompt
    @extend .light-box-shadow
    background-color: $color-dark
    border-radius: .5rem
    border-left-width: 5px
    border-left-style: solid
    color: $color-light
    left: 2rem
    padding: .5rem 1rem
    position: absolute
    top: 2rem
    z-index: 3

    &--origin
      border-left-color: $color-origin

    &--destination
      border-left-color: $color-destination

    @media #{map-get($display-breakpoints, 'sm-and-down')}
    @media #{map-get($display-breakpoints, 'md-and-up')}
      display: none

  @media #{map-get($display-breakpoints, 'sm-and-down')}
    grid-row-start: 2

  @media #{map-get($display-breakpoints, 'md-and-up')}
    grid-row-end: span 2

.map
  height: 100%
  position: absolute
  text-align: initial
  width: 100%
  z-index: 1

</style>

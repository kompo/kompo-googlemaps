<template>
    <vl-form-field v-bind="$_wrapperAttributes">
        <vlTaggableInput 
            v-bind="$_taggableInputAttributes" 
            v-on="$_taggableInputEvents"
            :labelKey="labelKey">
            <GmapAutocomplete
                class="vlFormControl"
                v-bind="$_attributes"
                v-on="$_events"
                type="search"
                ref="input" />
            <template v-slot:append>
                <i class="icon-location"/>
                <vl-link 
                    v-if="showPlaceEditLink"
                    class="place-editing"
                    :vkompo="placeEditLink"
                    :kompoid="kompoid"
                />
            </template>
        </vlTaggableInput>

        <GmapMap
            :center="center"
            :zoom="zoom"
            ref="gmap"
            map-type-id="terrain"
            style="height: 200px"
            @zoom_changed="setZoom"
            :options="mapOptions"
        >
            <div 
                v-for="(marker, index) in markers"
                :key=index
            >
                <component
                    v-if="marker.config"
                    :is=$_vueTag(marker)
                    :vkompo=marker
                    :kompoid=kompoid
                    :center="center"
                    @markerClicked="center = parseComponentPosition(marker)"
                />
                <GmapMarker
                    v-else
                    :position="marker"
                    :clickable="true"
                    :draggable="true"
                    :icon="marker.icon"
                    @click=" center = marker"
                />
            </div>
        </GmapMap>
    </vl-form-field>

</template>

<script>
import Field from 'vue-kompo/js/form/mixins/Field'
import HasTaggableInput from 'vue-kompo/js/form/mixins/HasTaggableInput'
import SetInitialValueAsArray from 'vue-kompo/js/form/mixins/SetInitialValueAsArray'
import draggable from 'vuedraggable'
import MarkerTrigger from './MarkerTrigger'

import * as VueGoogleMaps from 'vue2-google-maps'
Vue.use(VueGoogleMaps, {
  load: {
    key: process.env.MIX_GOOGLE_MAPS_API_KEY,
    libraries: 'places'
  }
})

export default {
    mixins: [Field, VueGoogleMaps, HasTaggableInput, SetInitialValueAsArray],
    components: { draggable, MarkerTrigger },

    data: () => ({
        labelKey: 'address',
        center: {lat:10, lng:10},
        zoom: 4,
        markers: [],
        placeEditLink: null,
    }),
    created(){
        if(this.defaultCenter){
            this.center = this.defaultCenter
            this.zoom = this.defaultZoom
        }

        this.updateEditLinkPayload()

        this.attachEditEvent()

        this.labelKey = this.formattedLabel
    },
    mounted(){
        if(this.$_value.length){
            this.$_value.forEach( (place) => {
                if(place.lat && place.lng)
                    this.setLocationWithMarkers({ lat: parseFloat(place.lat), lng: parseFloat(place.lng) })
            })
        }

        if (this.$_noAutocomplete) {
            setTimeout(() => this.$refs.input.$el.setAttribute('autocomplete', 'off-'+this.$_elementId()), 500)
        }
    },

    computed: {
        defaultCenter(){
            return this.$_config('defaultCenter')
        },
        defaultZoom(){
            return this.$_config('defaultZoom')
        },
        componentRestrictions(){ return this.$_config('componentRestrictions') },
        $_attributes() {
            return {
                ...this.$_defaultFieldAttributes,
                value: this.inputValue,
                options: Object.assign({
                    
                    },
                    this.componentRestrictions ? {
                        componentRestrictions: this.componentRestrictions
                    } : {}
                )
            }
        },
        $_events() {
            return {
                ...this.$_defaultFieldEvents,
                blur: this.blur,
                'place_changed': this.placeChanged
            }
        },
        $_pristine() {
            return this.$_value.length == 0
        },
        mapOptions(){
            return {
                disableDefaultUI: this.$_config('noDefaultUi') ? true : false,
                styles: this.$_config('customMapStyle') || GoogleMapsStyle || [],
            }
        },
        addMarkers(){
            return this.$_config('addMarkers')
        },
        initialPlaceEditLink(){ return this.$_config('placeEditLink') },
        showPlaceEditLink(){ return this.initialPlaceEditLink && this.$_value.length },
        editEventId(){ return 'vlUpdatePlace' + this.$_elKompoId },

        formattedLabel(){ return this.$_config('formattedLabel') || 'address' },
    },

    methods: {
        blur(){
            this.$_emptyInput()
            this.$_blurAction()
        },

        addRow(index) {
            this.component.value.push(_.cloneDeep(this.emptyRow))
        },

        deleteRow(index){
            if(this.$_value.length > 1){
                this.component.value.splice( index, 1)
            }else{
                this.component.value = [_.cloneDeep(this.emptyRow)]
            }
        },
        
        placeChanged(place) {
            place[this.formattedLabel] = place.formatted_address

            if(this.$_multiple){
                this.component.value.push(place)
            }else{
                this.component.value = [place]

                this.updateEditLinkPayload()
            }

            this.setLocationWithMarkers({
                lat: place.geometry.location.lat(),
                lng: place.geometry.location.lng()
            });
        },

        setLocationWithMarkers(place){
            this.setLocation(place)
            this.$_blurAction()

            if (this.addMarkers) {
                let closestMarkers = this.findClosestElements(this.addMarkers, place)
                this.placeMarkersOnMap(closestMarkers)
            }

            if(this.markers.length)
                this.centerAndZoom()
        },
        placeMarkersOnMap(markers){
            markers.forEach( (place) => {
                const pos = this.getPosition(place);

                if(pos.lat && pos.lng){
                    this.markers.push(place);
                }
            })
        },
        getDistanceLatLng(position1,position2) {
            var lat1=this.getPosition(position1).lat;
            var lat2=this.getPosition(position2).lat;
            var lon1=this.getPosition(position1).lng;
            var lon2=this.getPosition(position2).lng;
            function deg2rad(deg) {
                return deg * (Math.PI/180)
            }
            const R = 6371; // Radius of the earth in kilometers
            const dLat = deg2rad(lat2 - lat1);  // deg2rad below
            const dLon = deg2rad(lon2 - lon1); 
            const a = 
                Math.sin(dLat/2) * Math.sin(dLat/2) +
                Math.cos(deg2rad(lat1)) * Math.cos(deg2rad(lat2)) * 
                Math.sin(dLon/2) * Math.sin(dLon/2)

            const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a)); 
            const d = R * c; // Distance in km
            return d;
        },
        findClosestElements(array, initialValue) {
            // Sort the array based on the absolute difference from the initial value
            array.sort((a, b) => this.getDistanceLatLng(a, initialValue) - this.getDistanceLatLng(b, initialValue));
            
            // Select the first five elements
            return array.slice(0, 5);
        },
        $_bestGuessLabelFromSelection(selection){
            return selection.address
        },        
        $_fill(jsonFormData) {
            this.$_value.forEach( (place, i) => {
                jsonFormData[this.$_name + '[' + i + ']']= JSON.stringify(place)          
            })
        },
        $_remove(index) {
            HasTaggableInput.methods.$_remove.call(this, index)
            this.markers.splice( index, 1)
        },
        setLocation(location){
            if(this.$_multiple){
                this.markers.push(location)
            }else{
                this.markers = [location]
            }
            this.centerAndZoom()
        },
        centerAndZoom(){

            this.$refs.gmap.$mapPromise.then((map) => {
                const bounds = new google.maps.LatLngBounds()
                for (let m of this.markers) {
                    const pos = this.getPosition(m);
                    if(pos.lat && pos.lng)
                        bounds.extend(new google.maps.LatLng(pos.lat, pos.lng))
                }
                if(this.markers.length <= 1){
                    map.setCenter(bounds.getCenter())
                    map.setZoom(14) //setZoom doesnt work after fitBounds... so had to separate it
                }else{
                    map.fitBounds(bounds);
                }
            });
        },
        setZoom(zoom){
            this.zoom = zoom
        },
        updateEditLinkPayload(){
            if (!this.initialPlaceEditLink) {
                return
            }

            let withUpdatedPayload = this.initialPlaceEditLink
            let place = this.component.value[0]

            if (!place) {
                return
            }

            if (place.address_components) {
                place.address_components.forEach((item) => {

                    if (item.types[0] == 'postal_code') {
                        place.postal_code = item.long_name 
                    }

                    if (item.types[0] == 'locality') {
                        place.city = item.long_name 
                    }
                })

                place.address = place.name
                
            }

            withUpdatedPayload.interactions[0].action.config.ajaxPayload = Object.assign(place, {event: this.editEventId})
            this.placeEditLink = Object.assign({}, withUpdatedPayload)
        },
        attachEditEvent(){ //did it in create because it was loading 4 times using regular channels..
            if (!this.initialPlaceEditLink) {
                return
            }

            this.$_vlOn(this.editEventId, (payload) => {

                let place = this.$_value[0]
                place.address = payload.address;
                place.postal_code = payload.postal_code;
                place.city = payload.city;
                place[this.formattedLabel] = payload.address+' '+payload.postal_code+' '+payload.city

                this.component.value = [place]
                this.updateEditLinkPayload()
                
            })
        },

        getPosition(marker) {
            if (marker.config) {
                return this.parseComponentPosition(marker)
            }

            return marker
        },

        parseComponentPosition(component){
            return {
                lat: parseFloat(component.config.lat),
                lng: parseFloat(component.config.lng),
            }
        },
    }
}
</script>
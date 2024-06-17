<template>
    <GmapMarker
        :position="position"
        :clickable="true"
        :draggable="draggable"
        :icon="icon"
        v-bind="$_attributes"
        @click="handleMarkerClick"
    />
</template>

<script>
import Trigger from 'vue-kompo/js/form/mixins/Trigger'

export default {
    mixins: [Trigger],
    computed:{
        $_attributes(){
            return {
                ...this.$_defaultTriggerAttributes,
            }
        },
        draggable(){
            return this.$_config('draggable');
        },
        icon(){
            return this.$_config('icon');
        },
        position(){
            return {
                lat: parseFloat(this.$_config('lat')),
                lng: parseFloat(this.$_config('lng')),
            }
        },
    },
    methods: {
        handleMarkerClick(){
            this.$emit('markerClicked', this.position);
            
            this.$_clickAction();
        }
    },
    props: {
        center: Object,
    }
}
</script>

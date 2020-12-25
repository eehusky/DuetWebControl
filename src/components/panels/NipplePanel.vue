<style>
.move-btn {
    padding-left: 0 !important;
    padding-right: 0 !important;
    min-width: 0;
}

.nipplepad {
  
  position: relative;
  width: 150px;
  height: 150px;
    margin: auto;
}


</style>

<template>
    <v-card>
        <v-card-title>
            <code-btn v-show="visibleAxes.length" color="primary" small code="G28" :title="$t('button.home.titleAll')" class="ml-0 hidden-sm-and-down">
                {{ $t('button.home.captionAll') }}
            </code-btn>
            <v-spacer class="hidden-sm-and-down"></v-spacer>
            <v-icon small class="mr-1">mdi-swap-horizontal</v-icon> {{ $t('panel.movement.caption') }}
            <v-spacer></v-spacer>
        </v-card-title>

        <v-card-text>
            <v-row>
                <v-col>

                        <div  ref="nippleRef" class="nipplepad"></div>
                </v-col>
            </v-row>

        </v-card-text>

        <v-alert :value="unhomedAxes.length !== 0" type="warning" class="mb-0">
            {{ $tc('panel.movement.axesNotHomed', unhomedAxes.length) }}
            <strong>
                {{ unhomedAxes.map(axis => axis.letter).join(', ') }}
            </strong>
        </v-alert>

        <v-alert :value="visibleAxes.length === 0" type="info">
            {{ $t('panel.movement.noAxes') }}
        </v-alert>
    </v-card>
</template>

<script>
'use strict'

import { mapState, mapGetters, mapActions, mapMutations } from 'vuex'

import { KinematicsName } from '../../store/machine/modelEnums.js'
import nipplejs from 'nipplejs'

export default {
    computed: {
        ...mapGetters(['isConnected', 'uiFrozen']),
        ...mapState('machine/model', ['move']),
        ...mapState('machine/settings', ['moveFeedrate']),
        ...mapGetters('machine/settings', ['moveSteps', 'numMoveSteps']),
        isCompensationEnabled() { return this.move.compensation.type.toLowerCase() !== 'none' },
        visibleAxes() { return this.move.axes.filter(axis => axis.visible); },
        isDelta() {
            return ((this.move.kinematics.name === KinematicsName.delta) ||
                    (this.move.kinematics.name === KinematicsName.rotaryDelta));
        },
        unhomedAxes() { return this.move.axes.filter(axis => axis.visible && !axis.homed); }
    },
    mounted(){
            // eslint-disable-next-line no-unused-vars
            const staticGamepad = nipplejs.create({
              //zone: document.querySelector('.nipplepad'),
              zone: this.$refs.nippleRef,
              mode: 'static',
              position: { left: '50%', top: '50%' },
              color: '#c9c3b2',
              threshold: 0.25,
              fadeTime: 400,
              maxNumberOfNipples: 1,
              dynamicPage: true, // needed because of vue
              size: 150
            })
    },
    data() {
        return {
            showMeshEditDialog: false,
            moveStepDialog: {
                shown: false,
                axis: 'X',
                index: 0,
                preset: 0
            }
        }
    },
    methods: {
        ...mapActions('machine', ['sendCode']),
        ...mapMutations('machine/settings', ['setMoveStep']),

    },
    watch: {
        isConnected() {
            // Hide dialogs when the connection is interrupted
            this.showMeshEditDialog = false;
            this.moveStepDialog.shown = false;
        }
    }
}
</script>

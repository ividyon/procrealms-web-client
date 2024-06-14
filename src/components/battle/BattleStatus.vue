<template>
  <div class="battle-area">
    <div class="battle-status">
      <template v-for="side in ['good', 'vs', 'evil']">
        <div v-if="side === 'vs'" class="vs">VS</div>
        <div v-if="side !== 'vs'" :class="getSideClass(side)">
          <div class="side-scroll">
            <div v-for="participant in getParticipants(side)" class="entity" :key="participant.eid" :id="'entity-' + participant.eid">
              <TransitionGroup appear name="damage">
                <div style="left: 10px" class="damage damage-enter-active">532</div>
                <div
                    v-for="(anim, i) in state.animations.filter(a => a.eid == participant.eid && a.type == 'damage')"
                    :key="anim.key"
                    :style="{ left: getDamagePosition(participant.eid, i) }"
                    :class="getAnimationClass(anim)"
                >{{ anim.amount }}
                </div>
              </TransitionGroup>

              <TransitionGroup appear name="healing">
                <div
                    v-for="(anim, i) in state.animations.filter(a => a.eid == participant.eid && a.type == 'healing')"
                    :key="anim.key"
                    :style="{ left: `${10 + i * 50}px` }"
                    :class="getAnimationClass(anim)"
                >{{ anim.amount }}
                </div>
              </TransitionGroup>

              <BattleEntity :entity="getPartyEntity(participant)" :participant="participant"
                            :side="side" />
            </div>
          </div>
        </div>
      </template>
    </div>
  </div>

</template>

<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue'

import { state } from '@/static/state'
import { useHelpers } from '@/composables/helpers'

import BattleEntity from '@/components/battle/BattleEntity.vue'

const { selectNearestElement } = useHelpers()

let selectedElement = null

function selectBattleAction (degree) {
  selectedElement = selectNearestElement(selectedElement, degree)
}

function performBattleAction () {
  if (selectedElement) {
    selectedElement.click()
  }
}

function getDamagePosition(eid, i) {
  const element = document.getElementById(`participant-${eid}`);
  console.debug(element);
  return `${element.left + 10 + (i * 50)}px`;
}

function getParticipants (side) {
  let participants = Object.values(state.gameState.battle.participants).filter(p => p.side == side)
  participants.sort((a, b) => a.sort < b.sort ? -1 : 1)
  return participants
}

function getParticipantsPerRow () {
  var m = window.matchMedia('(max-width: 1075px)')
  return !m.matches ? 3 : 2 // 2 per row on mobile
}

// Gets the participants of a side which belong to a certain row.
// Row begins at 1.
function getSideInRow (side, row) {
  const participants = getParticipants(side)
  const perRow = getParticipantsPerRow()

  const startIndex = (row * perRow) - perRow
  const endIndex = (row * perRow)

  return participants.slice(startIndex, Math.min(endIndex, participants.length)) // Clamp by amount of participants.
}

// Gets the amount of participant rows per side.
// Ex. 7 participants = 3 rows (3, 3, 2).
function getRows (side) {
  const perRow = getParticipantsPerRow()

  const len = getParticipants(side).length
  const rows = Math.ceil(len / perRow)

  return Math.max(1, rows) // To ward off against 3 participants returning 0 rows.
}

function getPartyEntity (participant) {
  return state.gameState.party[participant.eid]
}

function getAnimationClass (anim) {
  let classes = [anim.type]
  if (anim.crit) {
    classes.push('crit')
  }
  return classes.join(' ')
}

function getSideClass (side) {
  let classes = ['side']
  classes.push(side)
  return classes.join(' ')
}

onMounted(() => {
  state.inputEmitter.on('selectBattleAction', selectBattleAction)
  state.inputEmitter.on('performBattleAction', performBattleAction)
})

onBeforeUnmount(() => {
  state.inputEmitter.off('selectBattleAction', selectBattleAction)
  state.inputEmitter.off('performBattleAction', performBattleAction)
})
</script>

<style lang="less" scoped>
@side-width: 50%;
@side-distance: 50px;
@side-fade-width: 100px;

.battle-area {
  margin-bottom: 10px;
  border-bottom: 2px solid #333;
}

.battle-status {
  display: flex;
  width: 100%;
  flex-direction: column-reverse;
  align-items: stretch;
  overflow-y: visible;

  .vs {
    z-index: 1;
    padding: 0;
    margin: 0 4px;
    flex: 1 1 0;
    position: relative;
    font-family: 'DOS', monospace;
    animation: vs 0.4s ease-in forwards;
    font-size: 1.1rem;
    text-align: center;
    letter-spacing: 2px;
    //color: #c50f1f;
    color: #f5f5f5;
    //text-shadow:
    //      0px -2px 2px black,
    //      2px 0px 2px black,
    //      -2px 0px 2px black,
    //      0px -4px 4px #fff,
    //      0px -4px 4px #fff,
    //  0px -6px 6px #FF3,
    //  0px -6px 6px #FF3,
    //  0px -8px 8px #F90,
    //  0px -8px 8px #F90,
    //  0px -16px 12px #C33,
    //  0px -16px 12px #C33;
    text-shadow: 0px -2px 2px black,
    2px 0px 2px black,
    -2px 0px 2px black;

    &::before, &::after {
      animation: vsburn 0.6s ease-out 0.2s forwards;
      opacity: 0;
      z-index: 0;
      position: absolute;
      color: transparent;
      bottom: 0;
      left: 50%;
      transform: translateX(-50%);
      content: 'VS';
      text-shadow: none;
    }

    &::before {
      z-index: -1;
      font-size: 1.25em;
      text-shadow: 0 0 0 rgba(#ffb700, 0.7),
      0 0 8px #F90;
    }

    &::after {
      bottom: -2px;
      z-index: -2;
      font-size: 1.6em;
      text-shadow: 0 0 0 rgba(#C33, 0.5),
      0 0 8px #C33;
    }
  }

  .side {
    display: flex;
    position: relative;
    padding: 10px 0;
    overflow-y: visible;
    &:before, &:after {
      position: absolute;
      top: 0;
      bottom: 0;
      width: @side-fade-width;
      pointer-events: none;
    }
    &:before {
      left: 0;
      background: linear-gradient(90deg, @color--body, transparent);
    }
    &:after {
      right: 0;
      background: linear-gradient(270deg, @color--body, transparent);
    }

    .side-scroll {
      box-sizing: border-box;
      padding: 0;
      display: flex;
      overflow-x: auto;
      overflow-y: visible;
      gap: 10px;
      flex: 0 0 calc(@side-width + @side-fade-width + @battle-entity-width / 2);
      flex-wrap: nowrap;
      width: 0;
    }

    &.good {
      padding-top: @side-distance;
      justify-content: start;
      .side-scroll {
        padding: 0 (@side-fade-width + 4px) 0 0;
        justify-content: right;
        flex-direction: row-reverse;
      }
      &:after {
        content: '';
        right: calc(100% - calc(@side-width + @side-fade-width + @battle-entity-width / 2));
      }
    }

    &.evil {
      padding-bottom: @side-distance;
      justify-content: end;
      .side-scroll {
        padding: 0 0 0 (@side-fade-width + 4px);
        justify-content: left;
        flex-direction: row;
      }
      &:before {
        content: '';
        left: calc(100% - calc(@side-width + @side-fade-width + @battle-entity-width / 2));
      }
    }

    .entity {
      overflow: visible;

      .damage {
        position: absolute;
        top: 0;
        font-size: 1.4rem;
        color: #ff3333;
        padding: 5px 10px;
        opacity: 0.8;
        font-weight: bold;
        .stroke(black, 2px);
        //background-color: #101014;
        z-index: 10;

        &.crit {
          line-height: 1.2rem;
          font-size: 1.9rem;
          color: #ffcc33;
        }

        &.damage-enter-active {
          animation: damage 2s ease-out infinite;
        }
      }

      .healing {
        position: absolute;
        top: -40px;
        font-size: 1.4rem;
        color: #33ff33;
        padding: 5px 10px;
        opacity: 0.8;
        background-color: #101014;
        z-index: 10;

        .healing-enter-active {
          animation: healing 2s ease-out forwards;
        }
      }
    }

    &.evil {
      .entity {
        .damage {
          top: auto;
          bottom: 0;
          &.damage-enter-active {
            animation-name: damage-down;
          }
        }

        .healing {
          top: auto;
          bottom: -40px;
          &.damage-enter-active {
            animation-name: healing-down;
          }
        }
      }
    }
  }
}

@keyframes damage {
  0% {
    opacity: 1;
    top: 0;
  }
  60% {
    opacity: 0.9;
    top: -35px;
  }
  100% {
    opacity: 0;
    top: -40px;
  }
}

@keyframes healing {
  0% {
    opacity: 1;
    top: -40px;
  }
  60% {
    opacity: 0.9;
    top: -5px;
  }
  100% {
    opacity: 0;
    top: 0;
  }
}
@keyframes damage-down {
  0% {
    opacity: 1;
    bottom: 0;
  }
  60% {
    opacity: 0.9;
    bottom: -35px;
  }
  100% {
    opacity: 0;
    bottom: -40px;
  }
}

@keyframes healing-down {
  0% {
    opacity: 1;
    bottom: -40px;
  }
  60% {
    opacity: 0.9;
    bottom: -5px;
  }
  100% {
    opacity: 0;
    bottom: 0;
  }
}

@keyframes vs {
  0% {
    z-index: 2;
    rotate: 20deg;
    scale: 400%;
  }
  60% {
    scale: 80%;
    rotate: -10deg;
  }
  80% {
    scale: 110%;
  }
  99% {
    z-index: 2;
  }
  100% {
    z-index: auto;
    rotate: 0deg;
    scale: 100%;
  }
}

@keyframes vsburn {
  0% {
    opacity: 0;
    translate: 50% 0;
    scale: 200%;
  }
  100% {
    opacity: 1;
    translate: 0 0;
    scale: 100%;
  }
}

@media screen and (max-width: 1075px) {
  .battle-status {
    .game.show-mobile-menu & {
      flex-direction: column;
    }

    .side {
      flex-basis: auto;
      flex-direction: column-reverse;

      .side-row {
        flex-direction: row;
      }

      &.evil {
        flex-direction: column;

        .side-row {
          flex-direction: row;
        }
      }
    }
  }
}

@media screen and (max-width: 800px) {
  .battle-status {
    flex-direction: column;

    .side {
      flex-direction: column-reverse;

      .side-row {
        flex-direction: column-reverse;
      }

      &.evil {
        flex-direction: column;

        .side-row {
          flex-direction: column;
        }
      }
    }
  }
}

</style>

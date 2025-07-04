<script setup lang="ts">
import { ref, computed } from 'vue'
import WorldMapGame from '@/modules/map-renderer/WorldMapGame.vue'
import ExerciseInstruction from './ExerciseInstruction.vue'
import { useLearningData } from '@/modules/learning-content/data/useLearningData'
import { useLearningEventStore } from '@/modules/learning-content/tracking/learning-event/learningEventStore'
import type { LearningEvent } from '@/modules/learning-content/tracking/learning-event/LearningEvent'
import { useExerciseProgressStore } from '@/modules/learning-content/tracking/exercise/exerciseProgressStore'
import { useRemoteLogging } from '@/modules/learning-content/tracking/remote-logging/useRemoteLogging'

const { pickRandomExerciseFromPickedLearningGoal, updateLearningGoalProgress } = useLearningData()
const eventStore = useLearningEventStore()
const exerciseProgressStore = useExerciseProgressStore()
const { logLearningEvent } = useRemoteLogging()

const currentExerciseData = ref<ReturnType<typeof pickRandomExerciseFromPickedLearningGoal> | undefined>(pickRandomExerciseFromPickedLearningGoal())
const attempts = ref(0)
const isSuccess = ref(false)

const currentExercise = computed(() => currentExerciseData.value?.exercise)
const currentLearningGoal = computed(() => currentExerciseData.value?.learningGoal)

const instruction = computed(() => {
  if (!currentExercise.value) return ''
  return currentExercise.value.instruction
})

const handleGameComplete = async (result: LearningEvent) => {
  attempts.value = result.numberOfClicksNeeded
  isSuccess.value = true
  eventStore.addEvent(result)
  exerciseProgressStore.updateProgressFromEvent(result)

  // Update learning goal progress
  if (currentExercise.value) {
    updateLearningGoalProgress(currentExercise.value.id, result.numberOfClicksNeeded === 1, result)
    
    // Log to remote server with the EXACT learning goal that was used to pick this exercise
    if (currentLearningGoal.value) {
      await logLearningEvent(result, currentLearningGoal.value.name)
    }
  }

  setTimeout(() => {
    currentExerciseData.value = pickRandomExerciseFromPickedLearningGoal()
    isSuccess.value = false
    attempts.value = 0
  }, 1000)
}
</script>


<template>

  <div class="container mx-auto p-4">
    <div v-if="!currentExercise" class="flex justify-center items-center h-64 text-gray-400">
      No exercise available.
    </div>
    <div v-else class="relative">
      <Transition name="flip" mode="out-in">
        <div v-if="currentExercise" :key="currentExercise.id" 
          class="card card-border p-2 m-2 absolute top-4 left-1/2 -translate-x-1/2 z-10 bg-base-100/95 backdrop-blur-sm shadow-sm inline-block w-full max-w-4/5 text-center"
          :class="{ 'bg-success/95': isSuccess }"
        >
          <ExerciseInstruction :instruction="instruction" />
        </div>
      </Transition>

      <WorldMapGame
        v-if="currentExercise"
        :target-country-to-click="currentExercise.data.country"
        :zoom-level="currentExercise.data.zoom"
        :pan-index="currentExercise.data.panIndex"
        :exercise-id="currentExercise.id"
        @game-complete="handleGameComplete"
        :allow-more-than-one-attempt="true"
      />
    </div>
  </div>
</template>

<style scoped>
.flip-enter-active,
.flip-leave-active {
  transition: all 0.2s ease;
}

.flip-enter-from,
.flip-leave-to {
  opacity: 0;
  transform: perspective(400px) rotateX(90deg);
}

.flip-enter-to,
.flip-leave-from {
  opacity: 1;
  transform: perspective(400px) rotateX(0);
}
</style>

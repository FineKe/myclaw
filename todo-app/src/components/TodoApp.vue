<script setup lang="ts">
import { ref, computed, watch, onMounted } from 'vue'
import { Plus, CheckCircle2, Circle, Trash2, ListTodo } from 'lucide-vue-next'
import type { Todo, FilterType } from '../types'

const todos = ref<Todo[]>([])
const newTodo = ref('')
const filter = ref<FilterType>('all')

const STORAGE_KEY = 'friday-todos'

onMounted(() => {
  const saved = localStorage.getItem(STORAGE_KEY)
  if (saved) todos.value = JSON.parse(saved)
})

watch(todos, (val) => {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(val))
}, { deep: true })

const filteredTodos = computed(() => {
  switch (filter.value) {
    case 'active': return todos.value.filter(t => !t.completed)
    case 'completed': return todos.value.filter(t => t.completed)
    default: return todos.value
  }
})

const activeCount = computed(() => todos.value.filter(t => !t.completed).length)
const completedCount = computed(() => todos.value.filter(t => t.completed).length)

function addTodo() {
  const text = newTodo.value.trim()
  if (!text) return
  todos.value.unshift({
    id: crypto.randomUUID(),
    text,
    completed: false,
    createdAt: Date.now(),
  })
  newTodo.value = ''
}

function toggleTodo(id: string) {
  const todo = todos.value.find(t => t.id === id)
  if (todo) todo.completed = !todo.completed
}

function removeTodo(id: string) {
  todos.value = todos.value.filter(t => t.id !== id)
}

function clearCompleted() {
  todos.value = todos.value.filter(t => !t.completed)
}

const filters: { label: string; value: FilterType }[] = [
  { label: '全部', value: 'all' },
  { label: '进行中', value: 'active' },
  { label: '已完成', value: 'completed' },
]
</script>

<template>
  <div class="max-w-xl mx-auto px-4 py-12">
    <!-- Header -->
    <div class="text-center mb-8">
      <div class="inline-flex items-center gap-2 mb-2">
        <ListTodo class="w-8 h-8 text-indigo-500" />
        <h1 class="text-3xl font-bold text-gray-900">Todo List</h1>
      </div>
      <p class="text-gray-500 text-sm">记录你的每一件事</p>
    </div>

    <!-- Input -->
    <form @submit.prevent="addTodo" class="mb-6">
      <div class="flex gap-2">
        <input
          v-model="newTodo"
          type="text"
          placeholder="添加新任务..."
          class="flex-1 px-4 py-3 rounded-xl border border-gray-200 bg-white shadow-sm
                 focus:outline-none focus:ring-2 focus:ring-indigo-500/40 focus:border-indigo-400
                 transition-all text-sm"
        />
        <button
          type="submit"
          class="px-4 py-3 bg-indigo-500 hover:bg-indigo-600 text-white rounded-xl shadow-sm
                 transition-all active:scale-95 flex items-center gap-1.5 text-sm font-medium"
        >
          <Plus class="w-4 h-4" />
          添加
        </button>
      </div>
    </form>

    <!-- Filters -->
    <div class="flex items-center gap-1 mb-4 bg-gray-100 rounded-lg p-1">
      <button
        v-for="f in filters"
        :key="f.value"
        @click="filter = f.value"
        :class="[
          'flex-1 px-3 py-1.5 rounded-md text-sm font-medium transition-all',
          filter === f.value
            ? 'bg-white text-indigo-600 shadow-sm'
            : 'text-gray-500 hover:text-gray-700'
        ]"
      >
        {{ f.label }}
      </button>
    </div>

    <!-- Todo List -->
    <div class="space-y-2">
      <TransitionGroup name="list">
        <div
          v-for="todo in filteredTodos"
          :key="todo.id"
          class="group flex items-center gap-3 px-4 py-3 bg-white rounded-xl border border-gray-100
                 shadow-sm hover:shadow-md transition-all"
        >
          <button @click="toggleTodo(todo.id)" class="shrink-0 transition-colors">
            <CheckCircle2
              v-if="todo.completed"
              class="w-5 h-5 text-green-500"
            />
            <Circle
              v-else
              class="w-5 h-5 text-gray-300 hover:text-indigo-400"
            />
          </button>
          <span
            :class="[
              'flex-1 text-sm transition-all',
              todo.completed ? 'line-through text-gray-400' : 'text-gray-700'
            ]"
          >
            {{ todo.text }}
          </span>
          <button
            @click="removeTodo(todo.id)"
            class="shrink-0 opacity-0 group-hover:opacity-100 transition-opacity
                   text-gray-400 hover:text-red-500"
          >
            <Trash2 class="w-4 h-4" />
          </button>
        </div>
      </TransitionGroup>

      <!-- Empty State -->
      <div
        v-if="filteredTodos.length === 0"
        class="text-center py-12 text-gray-400"
      >
        <ListTodo class="w-12 h-12 mx-auto mb-3 opacity-30" />
        <p class="text-sm">
          {{ filter === 'all' ? '暂无任务，添加一个吧！' : filter === 'active' ? '没有进行中的任务' : '没有已完成的任务' }}
        </p>
      </div>
    </div>

    <!-- Footer -->
    <div
      v-if="todos.length > 0"
      class="mt-4 flex items-center justify-between text-xs text-gray-400 px-1"
    >
      <span>{{ activeCount }} 项待完成</span>
      <button
        v-if="completedCount > 0"
        @click="clearCompleted"
        class="hover:text-red-500 transition-colors"
      >
        清除已完成
      </button>
    </div>
  </div>
</template>

<style scoped>
.list-enter-active,
.list-leave-active {
  transition: all 0.3s ease;
}
.list-enter-from {
  opacity: 0;
  transform: translateY(-10px);
}
.list-leave-to {
  opacity: 0;
  transform: translateX(20px);
}
</style>

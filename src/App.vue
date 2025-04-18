<template>
  <div class="todo-container">
    <div class="todo-header">
      <h1>Vue Todo App</h1>
    </div>
    <div class="todo-form">
      <input 
        type="text" 
        v-model="newTodo" 
        placeholder="タスクを入力..." 
        @keyup.enter="addTodo"
      />
      <select v-model="newTodoPriority">
        <option value="low">低</option>
        <option value="medium">中</option>
        <option value="high">高</option>
      </select>
      <button @click="addTodo">追加</button>
    </div>

    <!-- フィルタータブ -->
    <div class="todo-filters">
      <button 
        class="todo-filter-btn" 
        :class="{ active: filter === 'all' }" 
        @click="filter = 'all'"
      >
        全て
      </button>
      <button 
        class="todo-filter-btn" 
        :class="{ active: filter === 'active' }" 
        @click="filter = 'active'"
      >
        未完了
      </button>
      <button 
        class="todo-filter-btn" 
        :class="{ active: filter === 'completed' }" 
        @click="filter = 'completed'"
      >
        完了
      </button>
    </div>

    <ul class="todo-list">
      <li 
        v-for="(todo, index) in filteredTodos" 
        :key="index" 
        class="todo-item"
      >
        <input 
          type="checkbox" 
          v-model="todo.completed"
        />
        <div class="todo-item-content">
          <!-- 編集モードの場合はテキスト入力フィールドを表示 -->
          <input
            v-if="todo.editing"
            type="text"
            v-model="todo.editText"
            class="todo-edit-input"
            @keyup.enter="saveEdit(todo)"
            @keyup.esc="cancelEdit(todo)"
            ref="editInput"
          />
          <!-- 編集モードでない場合はテキストを表示 -->
          <span 
            v-else
            class="todo-item-text" 
            :class="{ 'todo-item-completed': todo.completed }"
          >
            {{ todo.text }}
          </span>
          <!-- 優先度バッジは編集モードでない場合のみ表示 -->
          <span 
            v-if="!todo.editing"
            class="todo-priority-badge"
            :class="`priority-${todo.priority}`"
          >
            {{ getPriorityLabel(todo.priority) }}
          </span>
        </div>
        <div class="todo-item-actions">
          <!-- 編集モードの場合は優先度選択・保存・キャンセルボタンを表示 -->
          <div v-if="todo.editing" class="edit-actions">
            <select 
              v-model="todo.priority"
              class="todo-priority-select"
              :title="'優先度を変更'"
            >
              <option value="low">低</option>
              <option value="medium">中</option>
              <option value="high">高</option>
            </select>
            <button 
              class="todo-item-save" 
              @click="saveEdit(todo)"
              title="保存"
            >
              保存
            </button>
            <button 
              class="todo-item-cancel" 
              @click="cancelEdit(todo)"
              title="キャンセル"
            >
              キャンセル
            </button>
          </div>
          <!-- 編集モードでない場合は編集・削除ボタンのみ表示 -->
          <div v-else class="normal-actions">
            <button 
              class="todo-item-edit" 
              @click="startEdit(todo)"
              title="編集"
            >
              編集
            </button>
            <button 
              class="todo-item-delete" 
              @click="removeTodo(todo)"
            >
              削除
            </button>
          </div>
        </div>
      </li>
      <!-- タスクがないときのメッセージ -->
      <li v-if="filteredTodos.length === 0" class="todo-empty">
        表示するタスクがありません
      </li>
    </ul>
    <div class="todo-stats" v-if="todos.length > 0">
      <p>完了: {{ completedCount }} / {{ todos.length }}</p>
    </div>
  </div>
</template>

<script>
export default {
  name: 'App',
  data() {
    return {
      newTodo: '',
      newTodoPriority: 'medium',
      todos: [],
      filter: 'all' // フィルターのデフォルト値: 'all', 'active', 'completed'
    }
  },
  computed: {
    completedCount() {
      return this.todos.filter(todo => todo.completed).length
    },
    sortedTodos() {
      // 優先度の重み付け
      const priorityWeight = {
        'high': 3,
        'medium': 2,
        'low': 1
      };
      
      // 優先度に基づいてソート（高→中→低）
      return [...this.todos].sort((a, b) => {
        return priorityWeight[b.priority] - priorityWeight[a.priority];
      });
    },
    filteredTodos() {
      // フィルターに基づいてタスクをフィルタリング
      switch(this.filter) {
        case 'active':
          return this.sortedTodos.filter(todo => !todo.completed);
        case 'completed':
          return this.sortedTodos.filter(todo => todo.completed);
        default:
          return this.sortedTodos;
      }
    }
  },
  methods: {
    addTodo() {
      if (this.newTodo.trim()) {
        this.todos.push({
          text: this.newTodo.trim(),
          completed: false,
          priority: this.newTodoPriority,
          editing: false,
          editText: ''
        })
        this.newTodo = ''
      }
    },
    removeTodo(todo) {
      const index = this.todos.indexOf(todo);
      if (index !== -1) {
        this.todos.splice(index, 1);
      }
    },
    getPriorityLabel(priority) {
      switch(priority) {
        case 'low': return '低';
        case 'medium': return '中';
        case 'high': return '高';
        default: return '中';
      }
    },
    startEdit(todo) {
      // 現在編集中のタスクがあれば編集をキャンセル
      this.todos.forEach(t => {
        if (t.editing) {
          this.cancelEdit(t);
        }
      });
      
      // 編集モードに入る
      todo.editText = todo.text;
      todo.editing = true;
      
      // 次のティックでテキストボックスにフォーカス
      this.$nextTick(() => {
        if (this.$refs.editInput && this.$refs.editInput[0]) {
          this.$refs.editInput[0].focus();
        }
      });
    },
    saveEdit(todo) {
      if (todo.editText.trim()) {
        todo.text = todo.editText.trim();
      }
      todo.editing = false;
      // 優先度が変更された可能性があるのでソートを更新
      this.sortTodos();
    },
    cancelEdit(todo) {
      todo.editing = false;
      todo.editText = todo.text;
    },
    sortTodos() {
      // 優先度が変更された時にちゃんと再レンダリングされるようにする
      // （計算プロパティは自動的に更新されるが、念のため）
      this.todos = [...this.todos];
    }
  }
}
</script>
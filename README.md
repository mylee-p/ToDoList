## ToDo List

<img width="1072" alt="todoList" src="https://user-images.githubusercontent.com/89143892/211495564-1b02d23f-a12e-4506-8bc5-889d3f1e9c1a.png">

-input에 할 일 입력 후 + 버튼을 누르면 할 일이 추가되는 기능<br/>-완료 표시, 수정 기능<br/>-삭제 기능 추가<br/>-필터 기능을 사용해서 전체, 해야될 것, 완료된 것을 구분<br/>-todoList에 추가한 내용이 LocalStorage에 저장되어 그대로 내용을 불러오는 기능 구현

### 개발환경설정 - Rollup.js

```
$ npm init -y
$ npm i -D rollup
$ npm i -D rollup-plugin-scss sass
$ npm i -D rollup-plugin-generate-html-template rollup-plugin-livereload
$ npm i -D rollup-plugin-serve rollup-plugin-terser
$ npm install --save @fortawesome/fontawesome-free
```

공통 설정

```js
// rollup.common.config.js

import htmlTemplate from 'rollup-plugin-generate-html-template';
import scss from 'rollup-plugin-scss';
import { nodeResolve } from '@rollup/plugin-node-resolve';
export default {
    input: 'src/js/index.js',
    output: {
        file: './dist/bundle.js',
        format: 'cjs',
        sourcemap: true,
    },
    plugins: [
        nodeResolve(),
        scss({
            insert: true,
            sourceMap: true,
        }),
        htmlTemplate({
            template: 'src/index.html',
            target: 'index.html',
        }),
    ],
};
```

dev server 설정

```js
// rollup.dev.config.js

import rollupCommonConfig from './rollup.common.config';
import serve from 'rollup-plugin-serve';
//watch가 돌았을 때 dev server를 다시 리로드 시켜주는 플러그인
import livereload from 'rollup-plugin-livereload';

const config = { ...rollupCommonConfig };

config.watch = {
    inclue: 'src/**',
};

config.plugins = [
    ...config.plugins,
    serve({
        host: 'localhost',
        port: 8080,
        open: true,
        contentBase: 'dist',
    }),
    livereload('dist'),
];

export default config;
```

```js
// rollup.prod.config.js

import rollupCommonConfig from './rollup.common.config';

//Uglify/Minify 설정만 넣어주면 됨
const config = { ...rollupCommonConfig };

config.plugins = [...config.plugins, terser()];

export default config;
```

ESLint / prettier 설정

```
$ npm i -D eslint
$ npm i -D --save-exact prettier
$ npm i -D eslint-config-prettier eslint-plugin-prettier
```

### HTML 구조

```html
<!-- index.html -->

<!DOCTYPE html>
<html>
    <head>
        <meta charset="UTF-8" />
        <title>TodoList</title>
        <link
            rel="shortcut icon"
            href="data:image/x-icon"
            type="image/x-icon"
        />
    </head>

    <body>
        <header>
            <h1>Todo List</h1>
        </header>
        <div class="input-container" id="input-container">
            <div class="input-area" id="input-area">
                <input
                    type="text"
                    class="todo-input"
                    id="todo-input"
                    maxlength="50"
                />
                <button class="todo-btn" id="add-btn" type="button">
                    <i class="fas fa-plus"></i>
                </button>
            </div>
            <div class="radio-area" id="radio-area">
                <input
                    type="radio"
                    id="filter1"
                    name="filter"
                    value="ALL"
                    checked
                />
                <label for="filter1">All</label>
                <input type="radio" id="filter2" name="filter" value="TODO" />
                <label for="filter2">Todo</label>
                <input type="radio" id="filter3" name="filter" value="DONE" />
                <label for="filter3">Done</label>
            </div>
        </div>
        <div class="todo-container" id="todo-container">
            <div class="todo-list" id="todo-list"></div>
        </div>
    </body>
</html>
```

### 할 일 생성 기능 구현

```js
// index.js

class TodoList {
    constructor() {
        this.assignElement();
        this.addEvent();
    }
    assignElement() {
        this.inputContainerEl = document.getElementById('input-container');
        this.inputAreaEl = this.inputContainerEl.querySelector('#input-area');
        this.todoInputEl = this.inputAreaEl.querySelector('#todo-input');
        this.addBtnEl = this.inputAreaEl.querySelector('#add-btn');
        this.todoContainerEl = document.getElementById('todo-container');
        this.todoListEl = this.todoContainerEl.querySelector('#todo-list');
    }
    // 이벤트를 추가
    addEvent() {
        //할 일 입력 후 add 버튼을 누르면 그 값을 가져와서 아래에 todoEl를 생성
        this.addBtnEl.addEventListener('click', this.onClickAddBtn.bind(this));
    }
    onClickAddBtn() {
        if (this.todoInputEl.value.length === 0) {
            alert('내용을 입력해주세요.');
            return;
        }
        this.createTodoElement(this.todoInputEl.value);
    }

    createTodoElement(value) {
        const todoDiv = document.createElement('div');
        todoDiv.classList.add('todo');

        //input태그를 넣어서 수정에 용이하도록 한다.
        const todoContent = document.createElement('input');
        todoContent.value = value;

        //수정버튼을 눌렀을 때만 수정할 수 있도록 하기
        todoContent.readOnly = true;
        todoContent.classList.add('todo-item');

        //버튼들 순서대로 만들기 [완료,수정,삭제]
        const fragment = new DocumentFragment();
        fragment.appendChild(todoContent);
        fragment.appendChild(
            this.createButton('complete-btn', 'complete-btn', [
                'fas',
                'fa-check',
            ]),
        );
        fragment.appendChild(
            this.createButton('edit-btn', 'edit-btn', ['fas', 'fa-edit']),
        );
        fragment.appendChild(
            this.createButton('delete-btn', 'delete-btn', ['fas', 'fa-trash']),
        );
        fragment.appendChild(
            this.createButton('save-btn', 'save-btn', ['fas', 'fa-save']),
        );
        todoDiv.appendChild(fragment);
        this.todoListEl.appendChild(todoDiv);
        //할 일을 추가한 후 input 내용을 지우는 것을 append한다.
        this.todoInputEl.value = '';
    }

    //버튼은 반복되는 작업이기 때문에 메소드를 이용해 따로 만들어준다
    createButton(btnId, btnClassName, iconClassName) {
        const btn = document.createElement('button');
        const icon = document.createElement('i');
        icon.classList.add(...iconClassName);
        btn.appendChild(icon);
        btn.id = btnId;
        btn.classList.add(btnClassName);
        return btn;
    }
}

//인스턴스 생성
document.addEventListener('DOMContentLoaded', () => {
    const todoList = new TodoList();
});
```

### 할 일 삭제 기능 구현

-이벤트 버블링 활용, todo-list 안에서 click을 캐치, 해당 버튼에 맞는 실행을 할 수 있도록 로직<br/>-이벤트를 한 번만 걸어서 나중에 동적으로 엘리먼트들이 추가되어도 따로 이벤트를 적용할 필요없는 로직

```js
// index.js

class TodoList {
    .
    .
    addEvent() {
        this.addBtnEl.addEventListener('click', this.onClickAddBtn.bind(this));
        //todo-list에 이벤트 적용
        this.todoListEl.addEventListener(
            'click',
            this.onClickTodoList.bind(this),
        );
    }

    onClickTodoList(event) {
        //Destructuring을 사용해서 이벤트 타겟을 뽑아내기
        const { target } = event;
        const btn = target.closest('button');
        if (btn.matches('#delete-btn')) {
            this.deleteTodo(target);
        }
    }

    //이벤트 타겟을 받기
    deleteTodo(target) {
        const todoDiv = target.closest('.todo');
        //todoDiv에 트랜지션이 끝났을 때의 시점을 캐치해서 지우는 이벤트를 추가
        todoDiv.addEventListener('transitionend', () => {
            //DOM구조에서 존재하는 엘리먼트 자체를 삭제
            todoDiv.remove();
        });
        //안보이게 지우기
        todoDiv.classList.add('delete');
    }
    .
    .
};
```

### 할 일 수정, 완료 기능 구현

```js
// index.js

class TodoList {
    .
    .
    onClickTodoList(event) {
        const { target } = event;
        const btn = target.closest('button');
        //버튼이 null일때 처리해주기
        if (!btn) return;

        if (btn.matches('#delete-btn')) {
            this.deleteTodo(target);
            //edit
        } else if (btn.matches('#edit-btn')) {
            this.editTodo(target);
            //save
        } else if (btn.matches('#save-btn')) {
            this.saveTodo(target)
            //complate
        } else if (btn.matches('#complete-btn')) {
            this.completeTodo(target);
        }
    }

    completeTodo(target) {
        const todoDiv = target.closest('.todo');
        //완료 버튼 클릭시 중간선 생성
        todoDiv.classList.toggle('done');

    }

    saveTodo(target) {
        const todoDiv = target.closest('.todo');
        //edit을 지우고
        todoDiv.classList.remove('edit');
        //todoInputEl을 찾아서 readonly를 해준다.
        const todoInputEl = todoDiv.querySelector('input');
        //save하면 수정할 수 없게 readonly를 true로 해준다.
        todoInputEl.readOnly = true
    }

    //수정버튼
    editTodo(target) {
        const todoDiv = target.closest('.todo');
        const todoInputEl = todoDiv.querySelector('input');
        //수정할거니까 입력이 가능하도록 해주기
        todoInputEl.readOnly = false;
        todoInputEl.focus();
        todoDiv.classList.add('edit');
    }
    .
    .
};
```

### 필터 기능 구현

-radio 버튼 탐색<br/>-all, todo, done 버튼을 눌렀을 때 value를 캐치해서 필터링 기능 구현<br/>-콜백을 실행하는 방식으로 필터를 실행하는 로직 구현

```js
//index.js

class TodoList {
    constructor() {
        this.assignElement();
        this.addEvent();
    }

    assignElement() {
        .
        .
        //radio-area 탐색
        this.radioAreaEl = this.inputContainerEl.querySelector('#radio-area');
        //radio 버튼들 탐색
        this.filterRadioBtnEls = this.radioAreaEl.querySelectorAll(
            'input[name="filter"]',
        );
    }
    .
    .
    //radio 버튼들에 이벤트를 적용
    addRadioBtnEvent() {
        for (const filterRadioBtnEl of this.filterRadioBtnEls) {
            filterRadioBtnEl.addEventListener(
                'click',
                this.onClickRadioBtn.bind(this),
            );
        }
    }

    onClickRadioBtn(event) {
        const { value } = event.target;
        console.log(value);
        this.filterTodo(value);
    }

    //필터 메소드는 따로  관리
    filterTodo(status) {
        const todoDivEls = this.todoListEl.querySelectorAll('div.todo');
        for (const todoDivEl of todoDivEls) {
            switch (status) {
                case 'ALL':
                    todoDivEl.style.display = 'flex';
                    break;
                case 'DONE':
                    todoDivEl.style.display = todoDivEl.classList.contains(
                        'done',
                    )
                        ? 'flex'
                        : 'none';
                    break;
                case 'TODO':
                    todoDivEl.style.display = todoDivEl.classList.contains(
                        'done',
                    )
                        ? 'none'
                        : 'flex';
                    break;
            }
        }
    }
    .
    .
};
```

### Router 개발

해시를 이용해서 구현

```js
// index.js

class Router {
    routes = [];
    notFoundCallback = () => {};
    addRoute(url, callback) {
        this.routes.push({
            url,
            callback,
        });
        return this;
    }
    checkRoute() {
        const currentRoute = this.routes.find(
            (route) => route.url === window.location.hash,
        );
        if (!currentRoute) {
            this.notFoundCallback();
            return;
        }
        currentRoute.callback();
    }
    init() {
        window.addEventListener('hashchange', this.checkRoute.bind(this));
        if (!window.location.hash) {
            window.location.hash = '#/';
        }
        this.checkRoute();
    }
    setNotFound(callback) {
        this.notFoundCallback = callback;
        return this;
    }
}
.
.
document.addEventListener('DOMContentLoaded', () => {
    const router = new Router();
    const todoList = new TodoList();
    const routeCallback = (status) => () => {
        todoList.filterTodo(status);
        document.querySelector(
            `input[type='radio'][value='${status}']`,
        ).checked = true;
    };
    router
        .addRoute('#/all', routeCallback('ALL'))
        .addRoute('#/todo', routeCallback('TODO'))
        .addRoute('#/done', routeCallback('DONE'))
        .setNotFound(routeCallback('ALL'))
        .init();
});
```

### LocalStorage에 할 일 data 연동

```js
// index.js

class Storage {
    saveTodo(id, todoContent) {
        const todosData = this.getTodos();
        todosData.push({ id, content: todoContent, status: 'TODO' });
        //만든 것을 다시 todos에 덮어씌워서 LocalStorage에 저장
        localStorage.setItem('todos', JSON.stringify(todosData));
    }
    editTodo() {}
    deleteTodo() {}
    getTodos() {
        //JSON.parse를 이용해서 기존데이터 가져오기
        return localStorage.getItem('todos') === null
            ? []
            : JSON.parse(localStorage.getItem('todos'));
    }
}

class TodoList {
    constructor(storage) {
        .
        .
        this.loadSavedData();
    }
    //storage 받아서 TodoList안에 storage 갖고 있는다.
    initStorage(storage) {
        this.storage = storage;
    }
    .
    .
    //로컬스토리지에 담긴 데이터 불러오기
    loadSavedData() {
        const todosData = this.storage.getTodos();
        for (const todoData of todosData) {
            const { id, content, status } = todoData;
            this.createTodoElement(id, content, status);
        }
    }
    .
    .
    onClickAddBtn() {
        if (this.todoInputEl.value.length === 0) {
            alert('내용을 입력해주세요.');
            return;
        }
        const id = Date.now();
        //LocalStorage에 데이터 추가
        this.storage.saveTodo(id, this.todoInputEl.value);
        this.createTodoElement(id, this.todoInputEl.value);
    }

    //id, value, status값을 받도록 수정
    createTodoElement(id, value, status = null) {
        const todoDiv = document.createElement('div');
        todoDiv.classList.add('todo');
        if (status === 'DONE') {
            todoDiv.classList.add('done');
        }

        //나중에 수정할 때 사용하기 위해 todoDiv쪽에 dataset속성을 사용해서 id 부여
        todoDiv.dataset.id = id;
        .
        .
    }
    .
    .
}

document.addEventListener('DOMContentLoaded', () => {
    const router = new Router();
    const todoList = new TodoList(new Storage());
    const routeCallback = (status) => () => {
        todoList.filterTodo(status);
        document.querySelector(
            `input[type='radio'][value='${status}']`,
        ).checked = true;
    };
    router
        .addRoute('#/all', routeCallback('ALL'))
        .addRoute('#/todo', routeCallback('TODO'))
        .addRoute('#/done', routeCallback('DONE'))
        .setNotFound(routeCallback('ALL'))
        .init();
});
```

### status 변경, edit, save도 LocalStorage에 연동

```js
// index.js

class Storage {
    saveTodo(id, todoContent) {
        const todosData = this.getTodos();
        todosData.push({ id, content: todoContent, status: 'TODO' });
        localStorage.setItem('todos', JSON.stringify(todosData));
    }
    editTodo(id, todoContent, status = 'TODO') {
        const todosData = this.getTodos();
        const todoIndex = todosData.findIndex((todo) => todo.id == id);
        const targetTodoData = todosData[todoIndex];
        const editedTodoData =
            todoContent === ''
                ? { ...targetTodoData, status }
                : { ...targetTodoData, content: todoContent };
        todosData.splice(todoIndex, 1, editedTodoData);
        localStorage.setItem('todos', JSON.stringify(todosData));
    }
    deleteTodo(id) {
        const todosData = this.getTodos();
        todosData.splice(
            todosData.findIndex((todo) => todo.id == id),
            1,
        );
        localStorage.setItem('todos', JSON.stringify(todosData));
    }
    getTodos() {
        return localStorage.getItem('todos') === null
            ? []
            : JSON.parse(localStorage.getItem('todos'));
    }
}

class TodoList {
    //클래스변수들 추가
    storage;
    inputContainerEl;
    inputAreaEl;
    todoInputEl;
    addBtnEl;
    todoContainerEl;
    todoListEl;
    radioAreaEl;
    filterRadioBtnEls;
    .
    .
    completeTodo(target) {
        const todoDiv = target.closest('.todo');
        todoDiv.classList.toggle('done');
        //storage.editTodo메소드를 불러와서 status 바꿔주기
        const { id } = todoDiv.dataset;
        this.storage.editTodo(
            id,
            '',
            todoDiv.classList.contains('done') ? 'DONE' : 'TODO',
        );
    }

    saveTodo(target) {
        const todoDiv = target.closest('.todo');
        todoDiv.classList.remove('edit');
        const todoInputEl = todoDiv.querySelector('input');
        todoInputEl.readOnly = true;
        const { id } = todoDiv.dataset;
        //editTodo에 id를 넣어주고 todoInputEl 넣어주기
        this.storage.editTodo(id, todoInputEl.value);
    }

    editTodo(target) {
        const todoDiv = target.closest('.todo');
        const todoInputEl = todoDiv.querySelector('input');
        todoInputEl.readOnly = false;
        todoInputEl.focus();
        todoDiv.classList.add('edit');
    }

    deleteTodo(target) {
        const todoDiv = target.closest('.todo');
        todoDiv.addEventListener('transitionend', () => {
            todoDiv.remove();
        });
        todoDiv.classList.add('delete');
        //todoDiv의 dataset의 id를 가져와서 삭제하기
        this.storage.deleleTodo(todoDiv.dataset.id);
    }
    .
    .
}
.
.
```

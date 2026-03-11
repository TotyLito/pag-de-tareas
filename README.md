<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Mi Enfoque - Gestor de Tareas</title>
    <!-- Tailwind CSS para el diseño -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- React y Babel para ejecutar el código en el navegador -->
    <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap');
        body { font-family: 'Inter', sans-serif; }
        .line-through { text-decoration: line-through; }
        .task-item { transition: all 0.2s ease-in-out; }
        @keyframes spin { from { transform: rotate(0deg); } to { transform: rotate(360deg); } }
        .animate-spin-custom { animation: spin 1s linear infinite; }
    </style>
</head>
<body class="bg-slate-50 text-slate-900">
    <div id="root"></div>

    <script type="text/babel" data-type="module">
        // Importar Firebase desde CDN
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-app.js";
        import { getAuth, signInAnonymously, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-auth.js";
        import { getFirestore, collection, onSnapshot, addDoc, updateDoc, deleteDoc, doc } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-firestore.js";

        // Configuración de entorno
        const firebaseConfig = JSON.parse(window.__firebase_config || '{}');
        const appId = window.__app_id || 'todo-app-local';

        // Inicialización de Firebase
        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);

        const { useState, useEffect, useMemo } = React;

        /**
         * Componente de Icono Seguro para React
         * Reemplaza la manipulación directa del DOM de Lucide por SVGs en línea 
         * para evitar el error 'removeChild' de React.
         */
        const Icon = ({ name, size = 20, className = "" }) => {
            const icons = {
                "plus": <path d="M12 5v14M5 12h14" />,
                "trash-2": <path d="M3 6h18M19 6v14a2 2 0 01-2 2H7a2 2 0 01-2-2V6m3 0V4a2 2 0 012-2h4a2 2 0 012 2v2" />,
                "check-circle-2": <path d="M22 11.08V12a10 10 0 11-5.93-9.14M22 4L12 14.01l-3-3" />,
                "circle": <circle cx="12" cy="12" r="10" />,
                "cloud": <path d="M17.5 19a3.5 3.5 0 11-5.83-2.66 7 7 0 1110.33-4.84A3.5 3.5 0 0117.5 19z" />,
                "alert-circle": <g><circle cx="12" cy="12" r="10"/><path d="M12 8v4M12 16h.01"/></g>,
                "tag": <path d="M12 2H2v10l9.29 9.29a1 1 0 001.42 0l8-8a1 1 0 000-1.42zM7 7h.01" />,
                "loader-2": <path d="M21 12a9 9 0 11-6.219-8.56" />,
                "check-square": <path d="M9 11l3 3L22 4M21 12v7a2 2 0 01-2 2H5a2 2 0 01-2-2V5a2 2 0 012-2h11" />
            };

            return (
                <svg 
                    width={size} 
                    height={size} 
                    viewBox="0 0 24 24" 
                    fill="none" 
                    stroke="currentColor" 
                    strokeWidth="2" 
                    strokeLinecap="round" 
                    strokeLinejoin="round" 
                    className={className}
                >
                    {icons[name] || null}
                </svg>
            );
        };

        const App = () => {
            const [tasks, setTasks] = useState([]);
            const [user, setUser] = useState(null);
            const [loading, setLoading] = useState(true);
            const [newTask, setNewTask] = useState('');
            const [newPriority, setNewPriority] = useState('medium');
            const [newCategory, setNewCategory] = useState('Personal');
            const [filter, setFilter] = useState('all');

            useEffect(() => {
                const initAuth = async () => {
                    try {
                        await signInAnonymously(auth);
                    } catch (e) { 
                        console.error("Error de autenticación:", e);
                        setLoading(false);
                    }
                };
                initAuth();
                const unsubscribe = onAuthStateChanged(auth, (u) => {
                    setUser(u);
                    if (!u) setLoading(false);
                });
                return () => unsubscribe();
            }, []);

            useEffect(() => {
                if (!user) return;
                const tasksRef = collection(db, 'artifacts', appId, 'users', user.uid, 'tasks');
                const unsubscribe = onSnapshot(tasksRef, 
                    (snapshot) => {
                        const data = snapshot.docs.map(d => ({ id: d.id, ...d.data() }));
                        setTasks(data);
                        setLoading(false);
                    }, 
                    (err) => {
                        console.error("Error al sincronizar:", err);
                        setLoading(false);
                    }
                );
                return () => unsubscribe();
            }, [user]);

            const priorities = {
                high: { label: 'Alta', color: 'text-red-500', bg: 'bg-red-50' },
                medium: { label: 'Media', color: 'text-amber-500', bg: 'bg-amber-50' },
                low: { label: 'Baja', color: 'text-emerald-500', bg: 'bg-emerald-50' }
            };

            const addTask = async (e) => {
                e.preventDefault();
                if (!newTask.trim() || !user) return;
                
                const tasksRef = collection(db, 'artifacts', appId, 'users', user.uid, 'tasks');
                try {
                    await addDoc(tasksRef, {
                        text: newTask,
                        priority: newPriority,
                        category: newCategory,
                        completed: false,
                        createdAt: Date.now()
                    });
                    setNewTask('');
                } catch (e) {
                    console.error("Error al guardar:", e);
                }
            };

            const toggleTask = async (id, status) => {
                if (!user) return;
                const ref = doc(db, 'artifacts', appId, 'users', user.uid, 'tasks', id);
                await updateDoc(ref, { completed: !status });
            };

            const deleteTask = async (id) => {
                if (!user) return;
                const ref = doc(db, 'artifacts', appId, 'users', user.uid, 'tasks', id);
                await deleteDoc(ref);
            };

            const filteredTasks = useMemo(() => {
                let list = tasks.filter(t => {
                    if (filter === 'active') return !t.completed;
                    if (filter === 'completed') return t.completed;
                    return true;
                });
                
                const order = { high: 0, medium: 1, low: 2 };
                return list.sort((a, b) => {
                    if (a.completed !== b.completed) return a.completed ? 1 : -1;
                    if (order[a.priority] !== order[b.priority]) return order[a.priority] - order[b.priority];
                    return (b.createdAt || 0) - (a.createdAt || 0);
                });
            }, [tasks, filter]);

            const stats = {
                total: tasks.length,
                completed: tasks.filter(t => t.completed).length,
                urgent: tasks.filter(t => t.priority === 'high' && !t.completed).length
            };

            if (loading) return (
                <div className="flex flex-col h-screen items-center justify-center gap-4">
                    <Icon name="loader-2" className="animate-spin-custom text-indigo-500" size={48} />
                    <p className="text-slate-400 font-medium animate-pulse">Sincronizando tareas...</p>
                </div>
            );

            return (
                <div className="max-w-3xl mx-auto p-4 md:p-8">
                    <header className="mb-8">
                        <div className="flex justify-between items-start mb-6">
                            <div>
                                <h1 className="text-3xl font-bold flex items-center gap-2 text-slate-800">
                                    Mi Enfoque <Icon name="cloud" className="text-indigo-500" />
                                </h1>
                                <p className="text-slate-500 mt-1">Organización persistente en la nube</p>
                            </div>
                            <div className="bg-white p-4 rounded-2xl shadow-sm border border-slate-100 flex gap-6">
                                <div className="text-center">
                                    <span className="block text-lg font-bold text-red-500 leading-none">{stats.urgent}</span>
                                    <span className="text-[10px] text-slate-400 font-bold uppercase tracking-wider">Urgentes</span>
                                </div>
                                <div className="w-px h-8 bg-slate-100"></div>
                                <div className="text-center">
                                    <span className="block text-lg font-bold text-emerald-500 leading-none">
                                        {stats.total ? Math.round((stats.completed / stats.total) * 100) : 0}%
                                    </span>
                                    <span className="text-[10px] text-slate-400 font-bold uppercase tracking-wider">Logrado</span>
                                </div>
                            </div>
                        </div>
                        <div className="w-full bg-slate-200 h-1.5 rounded-full overflow-hidden">
                            <div 
                                className="bg-indigo-500 h-full transition-all duration-700"
                                style={{ width: `${stats.total ? (stats.completed / stats.total) * 100 : 0}%` }}
                            ></div>
                        </div>
                    </header>

                    <form onSubmit={addTask} className="bg-white p-6 rounded-2xl shadow-sm border border-slate-200 mb-8 space-y-4 transition-all hover:shadow-md">
                        <div className="flex gap-2">
                            <input 
                                type="text" 
                                value={newTask} 
                                onChange={e => setNewTask(e.target.value)}
                                placeholder="¿Qué vas a resolver hoy?"
                                className="flex-1 bg-slate-50 rounded-xl px-4 py-3 outline-none focus:ring-2 focus:ring-indigo-500 border border-transparent focus:border-transparent transition-all"
                            />
                            <button className="bg-indigo-600 text-white p-3 rounded-xl hover:bg-indigo-700 transition-colors shadow-lg shadow-indigo-100">
                                <Icon name="plus" size={24} />
                            </button>
                        </div>
                        <div className="flex flex-wrap gap-4 text-sm font-medium">
                            <div className="flex items-center gap-2 bg-slate-50 px-3 py-1.5 rounded-lg border">
                                <Icon name="alert-circle" size={16} className="text-slate-400" />
                                <select value={newPriority} onChange={e => setNewPriority(e.target.value)} className="bg-transparent outline-none cursor-pointer">
                                    <option value="high">Prioridad Alta</option>
                                    <option value="medium">Prioridad Media</option>
                                    <option value="low">Prioridad Baja</option>
                                </select>
                            </div>
                            <div className="flex items-center gap-2 bg-slate-50 px-3 py-1.5 rounded-lg border">
                                <Icon name="tag" size={16} className="text-slate-400" />
                                <select value={newCategory} onChange={e => setNewCategory(e.target.value)} className="bg-transparent outline-none cursor-pointer">
                                    <option>Trabajo</option>
                                    <option>Personal</option>
                                    <option>Salud</option>
                                    <option>Hogar</option>
                                    <option>Urgente</option>
                                </select>
                            </div>
                        </div>
                    </form>

                    <div className="flex gap-4 mb-6 text-sm font-bold text-slate-400 border-b border-slate-200">
                        <button onClick={() => setFilter('all')} className={`pb-2 px-1 transition-all ${filter === 'all' ? 'text-indigo-600 border-b-2 border-indigo-600' : 'hover:text-slate-600'}`}>TODAS</button>
                        <button onClick={() => setFilter('active')} className={`pb-2 px-1 transition-all ${filter === 'active' ? 'text-indigo-600 border-b-2 border-indigo-600' : 'hover:text-slate-600'}`}>PENDIENTES</button>
                        <button onClick={() => setFilter('completed')} className={`pb-2 px-1 transition-all ${filter === 'completed' ? 'text-indigo-600 border-b-2 border-indigo-600' : 'hover:text-slate-600'}`}>COMPLETADAS</button>
                    </div>

                    <div className="space-y-3">
                        {filteredTasks.length === 0 ? (
                            <div className="text-center py-12 bg-white rounded-3xl border border-dashed border-slate-200">
                                <Icon name="check-square" size={48} className="mx-auto text-slate-100 mb-2" />
                                <p className="text-slate-400">No hay tareas en esta vista</p>
                            </div>
                        ) : (
                            filteredTasks.map(task => (
                                <div key={task.id} className={`task-item flex items-center gap-4 bg-white p-4 rounded-2xl border border-slate-100 shadow-sm ${task.completed ? 'bg-slate-50/50' : 'hover:border-indigo-100 hover:shadow-md'}`}>
                                    <button 
                                        onClick={() => toggleTask(task.id, task.completed)} 
                                        className={`transition-all transform active:scale-90 ${task.completed ? 'text-emerald-500' : 'text-slate-200 hover:text-indigo-400'}`}
                                    >
                                        <Icon name={task.completed ? "check-circle-2" : "circle"} size={26} />
                                    </button>
                                    <div className="flex-1 min-w-0">
                                        <div className="flex gap-2 mb-1 items-center">
                                            <span className={`text-[10px] font-bold px-2 py-0.5 rounded-full uppercase tracking-tighter ${priorities[task.priority].bg} ${priorities[task.priority].color}`}>
                                                {task.priority}
                                            </span>
                                            <span className="text-[10px] font-bold text-slate-400 bg-slate-100 px-2 py-0.5 rounded-full">
                                                {task.category}
                                            </span>
                                        </div>
                                        <p className={`text-slate-700 font-medium truncate ${task.completed ? 'line-through text-slate-400' : ''}`}>
                                            {task.text}
                                        </p>
                                    </div>
                                    <button 
                                        onClick={() => deleteTask(task.id)} 
                                        className="text-slate-200 hover:text-red-500 p-2 hover:bg-red-50 rounded-lg transition-all"
                                    >
                                        <Icon name="trash-2" size={18} />
                                    </button>
                                </div>
                            ))
                        )}
                    </div>
                    
                    <footer className="mt-12 pt-6 border-t border-slate-200 text-center">
                        <p className="text-[10px] text-slate-400 font-bold uppercase tracking-[0.2em]">ID de Sesión: {user?.uid}</p>
                    </footer>
                </div>
            );
        };

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>

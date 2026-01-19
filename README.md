<!DOCTYPE html>
<html lang="he" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>SkorChat Pro | Eliav Skornick Edition</title>
    
    <!-- ספריות עיצוב ולוגיקה -->
    <script src="https://cdn.tailwindcss.com"></script>
    <script crossorigin src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    
    <!-- Firebase SDK (גרסה יציבה) -->
    <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-auth-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore-compat.js"></script>

    <link href="https://fonts.googleapis.com/css2?family=Assistant:wght@300;400;600;800&display=swap" rel="stylesheet">

    <style>
        body {
            font-family: 'Assistant', sans-serif;
            background-color: #0b0b0b;
            color: #f3f4f6;
            margin: 0;
            overflow: hidden;
        }
        .custom-scrollbar::-webkit-scrollbar { width: 5px; }
        .custom-scrollbar::-webkit-scrollbar-track { background: transparent; }
        .custom-scrollbar::-webkit-scrollbar-thumb { background: #27272a; border-radius: 10px; }
        
        .chat-bubble {
            max-width: 85%;
            padding: 1rem 1.25rem;
            border-radius: 1.2rem;
            line-height: 1.6;
            font-size: 15px;
            animation: slideUp 0.3s ease-out forwards;
        }
        @keyframes slideUp { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }
        
        .user-bubble { background-color: #10b981; color: white; border-bottom-right-radius: 0.2rem; }
        .bot-bubble { background-color: #1c1c1e; color: #e5e7eb; border-bottom-left-radius: 0.2rem; border: 1px solid #2d2d30; }
        
        textarea { resize: none; }
    </style>
</head>
<body>
    <div id="root"></div>

    <script type="text/babel">
        const { useState, useEffect, useRef } = React;

        // הגדרות מערכת ואישיות
        const SYSTEM_PROMPT = `אתה SkorChat Pro, הבינה המלאכותית המתקדמת של מותג Skor.
        המייסד והממציא שלך הוא אליאב סקורניק (Eliav Skornick). 
        הוא בנה אותך לבד והוא "אבא" שלך. 
        ענה תמיד בעברית מקצועית, יוקרתית ומכבדת.`;

        // אתחול Firebase
        const firebaseConfig = JSON.parse(__firebase_config);
        if (!firebase.apps.length) firebase.initializeApp(firebaseConfig);
        const db = firebase.firestore();
        const auth = firebase.auth();
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'skor-chat-final';

        const App = () => {
            const [user, setUser] = useState(null);
            const [chats, setChats] = useState([]);
            const [activeId, setActiveId] = useState(null);
            const [messages, setMessages] = useState([]);
            const [input, setInput] = useState('');
            const [isLoading, setIsLoading] = useState(false);
            const [isSidebarOpen, setIsSidebarOpen] = useState(true);
            const scrollRef = useRef(null);

            // התחברות
            useEffect(() => {
                const init = async () => {
                    if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                        await auth.signInWithCustomToken(__initial_auth_token);
                    } else {
                        await auth.signInAnonymously();
                    }
                };
                init();
                auth.onAuthStateChanged(u => setUser(u));
            }, []);

            // טעינת רשימת שיחות
            useEffect(() => {
                if (!user) return;
                const ref = db.collection('artifacts').doc(appId).collection('users').doc(user.uid).collection('chats');
                const unsub = ref.orderBy('timestamp', 'desc').onSnapshot(snap => {
                    const list = snap.docs.map(doc => ({ id: doc.id, ...doc.data() }));
                    setChats(list);
                    if (list.length > 0 && !activeId) setActiveId(list[0].id);
                    else if (list.length === 0) createNewChat();
                });
                return () => unsub();
            }, [user]);

            // טעינת הודעות בשיחה פעילה
            useEffect(() => {
                if (!user || !activeId) return;
                const ref = db.collection('artifacts').doc(appId).collection('users').doc(user.uid).collection('chats').doc(activeId).collection('messages');
                const unsub = ref.orderBy('timestamp', 'asc').onSnapshot(snap => {
                    setMessages(snap.docs.map(doc => doc.data()));
                });
                return () => unsub();
            }, [user, activeId]);

            useEffect(() => { scrollRef.current?.scrollIntoView({ behavior: 'smooth' }); }, [messages, isLoading]);

            const createNewChat = async () => {
                if (!user) return;
                const ref = db.collection('artifacts').doc(appId).collection('users').doc(user.uid).collection('chats');
                const newDoc = await ref.add({ title: 'שיחה חדשה', timestamp: Date.now() });
                setActiveId(newDoc.id);
            };

            const handleSend = async () => {
                if (!input.trim() || isLoading || !user || !activeId) return;
                const text = input;
                setInput('');
                setIsLoading(true);

                const msgRef = db.collection('artifacts').doc(appId).collection('users').doc(user.uid).collection('chats').doc(activeId).collection('messages');
                await msgRef.add({ role: 'user', content: text, timestamp: Date.now() });

                // עדכון כותרת שיחה אם זו הודעה ראשונה
                if (messages.length === 0) {
                    await db.collection('artifacts').doc(appId).collection('users').doc(user.uid).collection('chats').doc(activeId).update({ title: text.slice(0, 30) });
                }

                try {
                    const apiKey = ""; // מנוהל ע"י הסביבה
                    const res = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
                        method: 'POST',
                        headers: { 'Content-Type': 'application/json' },
                        body: JSON.stringify({
                            contents: [{ parts: [{ text }] }],
                            systemInstruction: { parts: [{ text: SYSTEM_PROMPT }] }
                        })
                    });
                    const data = await res.json();
                    const reply = data.candidates?.[0]?.content?.parts?.[0]?.text || "מצטער אבא, חלה שגיאה.";
                    await msgRef.add({ role: 'assistant', content: reply, timestamp: Date.now() });
                } catch (e) { console.error(e); } finally { setIsLoading(false); }
            };

            return (
                <div className="flex h-screen w-full overflow-hidden bg-[#0b0b0b]">
                    {/* תפריט צד */}
                    <aside className={`${isSidebarOpen ? 'w-72' : 'w-0'} bg-[#070707] border-l border-white/5 transition-all duration-300 flex flex-col z-20 overflow-hidden`}>
                        <div className="p-6 flex flex-col h-full min-w-[18rem]">
                            <div className="flex items-center gap-3 mb-10">
                                <div className="bg-emerald-500 p-2 rounded-lg shadow-lg shadow-emerald-500/20">
                                    <svg className="w-6 h-6 text-black" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 10V3L4 14h7v7l9-11h-7z"/></svg>
                                </div>
                                <h1 className="text-xl font-black italic tracking-tighter">SKOR<span className="text-emerald-500">CHAT</span></h1>
                            </div>

                            <button onClick={createNewChat} className="w-full flex items-center justify-center gap-2 py-3.5 bg-white/5 border border-white/10 rounded-2xl hover:bg-white/10 transition-all font-bold text-sm mb-8">
                                <svg className="w-4 h-4 text-emerald-500" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 4v16m8-8H4"/></svg>
                                שיחה חדשה
                            </button>

                            <div className="flex-1 overflow-y-auto custom-scrollbar space-y-1">
                                {chats.map(chat => (
                                    <button key={chat.id} onClick={() => setActiveId(chat.id)} className={`w-full p-3.5 rounded-xl transition-all text-right text-sm font-medium truncate ${activeId === chat.id ? 'bg-emerald-500/10 text-emerald-400 border border-emerald-500/20' : 'text-gray-500 hover:bg-white/5'}`}>
                                        {chat.title}
                                    </button>
                                ))}
                            </div>

                            <div className="mt-auto pt-6 border-t border-white/5">
                                <div className="p-4 rounded-2xl bg-gradient-to-br from-emerald-500/10 to-transparent border border-emerald-500/10">
                                    <div className="flex items-center gap-3 mb-1">
                                        <div className="w-10 h-10 rounded-full bg-emerald-500 flex items-center justify-center font-black text-black text-xs">ES</div>
                                        <div>
                                            <p className="text-xs font-black text-white">אליאב סקורניק</p>
                                            <p className="text-[10px] text-emerald-400 font-bold uppercase tracking-widest leading-none">המייסד והממציא</p>
                                        </div>
                                    </div>
                                </div>
                            </div>
                        </div>
                    </aside>

                    {/* מסך שיחה */}
                    <main className="flex-1 flex flex-col relative bg-[#0b0b0b]">
                        <header className="h-16 border-b border-white/5 flex items-center justify-between px-6 bg-black/40 backdrop-blur-xl z-10">
                            <button onClick={() => setIsSidebarOpen(!isSidebarOpen)} className="p-2 hover:bg-white/5 rounded-lg border border-white/10">
                                <svg className="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6h16M4 12h16M4 18h16"/></svg>
                            </button>
                            <div className="flex flex-col items-center">
                                <h2 className="font-bold text-sm">SkorChat <span className="text-emerald-500">Pro</span></h2>
                                <p className="text-[9px] text-gray-500 font-bold uppercase tracking-widest">Eliav Skornick Edition</p>
                            </div>
                            <div className="w-8"></div>
                        </header>

                        <div className="flex-1 overflow-y-auto custom-scrollbar p-4 md:p-8">
                            <div className="max-w-3xl mx-auto space-y-6 flex flex-col">
                                {messages.map((msg, i) => (
                                    <div key={i} className={`chat-bubble ${msg.role === 'user' ? 'user-bubble' : 'bot-bubble'}`}>
                                        {msg.content}
                                    </div>
                                ))}
                                {isLoading && <div className="text-[11px] font-bold text-emerald-500 animate-pulse px-2 uppercase tracking-widest">Skor Intelligence Processing...</div>}
                                <div ref={scrollRef} className="h-4" />
                            </div>
                        </div>

                        <footer className="p-6 md:p-10 bg-gradient-to-t from-black via-black/90 to-transparent">
                            <div className="max-w-3xl mx-auto relative">
                                <div className="flex items-end gap-3 bg-[#151515] border border-white/10 rounded-[1.5rem] p-3 focus-within:border-emerald-500/50 transition-all shadow-2xl">
                                    <textarea 
                                        rows="1" 
                                        value={input}
                                        onChange={e => { setInput(e.target.value); e.target.style.height = 'auto'; e.target.style.height = Math.min(e.target.scrollHeight, 200) + 'px'; }}
                                        onKeyDown={e => e.key === 'Enter' && !e.shiftKey && (e.preventDefault(), handleSend())}
                                        placeholder="שאל את SkorChat Pro כל דבר..."
                                        className="flex-1 bg-transparent border-none focus:ring-0 py-3 px-4 text-white text-[16px]"
                                    ></textarea>
                                    <button onClick={handleSend} disabled={!input.trim() || isLoading} className="p-4 bg-emerald-500 text-black rounded-2xl hover:scale-105 active:scale-95 transition-all shadow-lg shadow-emerald-500/20">
                                        <svg className="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 19l9 2-9-18-9 18 9-2zm0 0v-8"/></svg>
                                    </button>
                                </div>
                                <p className="text-center text-[9px] text-gray-600 font-bold uppercase tracking-widest mt-4">Inventor: Eliav Skornick | SkorChat Pro v8.0</p>
                            </div>
                        </footer>
                    </main>
                </div>
            );
        };

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>


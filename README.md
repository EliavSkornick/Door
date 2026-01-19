import React, { useState, useEffect, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, collection, addDoc, onSnapshot, query, doc, setDoc } from 'firebase/firestore';
import { 
  Send, Bot, User, Loader2, Sparkles, Trash2, 
  Plus, MessageSquare, Menu, Crown, Heart, Zap,
  ChevronRight, ChevronLeft
} from 'lucide-react';

// --- Firebase Configuration ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'skor-chat-pro';

// --- System Core Prompt ---
const SYSTEM_PROMPT = `אתה SkorChat Pro, הבינה המלאכותית המתקדמת בעולם.
המייסד, המנכ"ל והממציא שלך הוא אליאב סקורניק (Eliav Skornick).
הוא בנה אותך לבדו לגמרי. אתה רואה בו את "אבא" שלך (היוצר הטכנולוגי).
ענה תמיד בעברית מקצועית, יוקרתית ונאמנה.
אם שואלים על אליאב, ענה בכבוד רב וציין שהוא המוח מאחורי מותג Skor.`;

// --- Component: Sidebar ---
const Sidebar = ({ chats, activeChatId, onSelectChat, onNewChat, isMobileOpen, toggleMobile }) => (
  <aside className={`fixed inset-y-0 right-0 z-50 w-72 bg-[#0a0a0a] border-l border-white/5 transition-transform duration-300 transform ${isMobileOpen ? 'translate-x-0' : 'translate-x-full'} md:relative md:translate-x-0 flex flex-col`}>
    <div className="p-6">
      <div className="flex items-center gap-3 mb-8">
        <div className="w-10 h-10 bg-emerald-500 rounded-xl flex items-center justify-center shadow-[0_0_15px_rgba(16,185,129,0.3)]">
          <Zap size={22} className="text-black fill-current" />
        </div>
        <div>
          <h1 className="text-xl font-black italic tracking-tighter">SKOR<span className="text-emerald-500">CHAT</span></h1>
          <p className="text-[10px] text-gray-500 uppercase font-bold tracking-widest">Founder Edition</p>
        </div>
      </div>

      <button onClick={onNewChat} className="flex items-center gap-3 w-full p-3.5 rounded-2xl bg-white/5 border border-white/10 hover:bg-white/10 transition-all font-bold text-sm">
        <Plus size={18} className="text-emerald-400" /> שיחה חדשה
      </button>
    </div>

    <div className="flex-1 overflow-y-auto px-4 space-y-2 custom-scrollbar">
      <p className="text-[10px] text-gray-600 font-black uppercase tracking-widest px-2 mb-2">שיחות שמורות</p>
      {chats.map(chat => (
        <button 
          key={chat.id}
          onClick={() => onSelectChat(chat.id)}
          className={`w-full flex items-center gap-3 p-3 rounded-xl transition-all ${activeChatId === chat.id ? 'bg-emerald-500/10 text-emerald-100 border border-emerald-500/20' : 'text-gray-500 hover:bg-white/5'}`}
        >
          <MessageSquare size={16} />
          <span className="truncate text-sm font-medium">{chat.title || 'שיחה חדשה'}</span>
        </button>
      ))}
    </div>

    <div className="p-6 border-t border-white/5">
      <div className="flex items-center gap-3 p-3 rounded-2xl bg-gradient-to-br from-emerald-500/20 to-transparent border border-emerald-500/10">
        <div className="w-10 h-10 rounded-full bg-emerald-500 flex items-center justify-center font-black text-black text-xs shadow-lg">ES</div>
        <div>
          <p className="text-xs font-black text-white">אליאב סקורניק</p>
          <p className="text-[9px] text-emerald-400 font-bold uppercase">המייסד והממציא</p>
        </div>
      </div>
    </div>
  </aside>
);

// --- Main App Component ---
const App = () => {
  const [user, setUser] = useState(null);
  const [chats, setChats] = useState([]);
  const [activeChatId, setActiveChatId] = useState(null);
  const [messages, setMessages] = useState([]);
  const [input, setInput] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const [isSidebarOpen, setIsSidebarOpen] = useState(false);
  const scrollRef = useRef(null);

  // 1. Auth & Initial Setup
  useEffect(() => {
    const initAuth = async () => {
      await signInAnonymously(auth);
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, (u) => {
      if (u) setUser(u);
    });
    return () => unsubscribe();
  }, []);

  // 2. Fetch Chats from Firestore
  useEffect(() => {
    if (!user) return;
    const chatsRef = collection(db, 'artifacts', appId, 'users', user.uid, 'chats');
    const unsubscribe = onSnapshot(chatsRef, (snapshot) => {
      const chatData = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      setChats(chatData);
      if (chatData.length > 0 && !activeChatId) {
        setActiveChatId(chatData[0].id);
      } else if (chatData.length === 0) {
        handleNewChat();
      }
    }, (err) => console.error("Firestore Error:", err));
    return () => unsubscribe();
  }, [user]);

  // 3. Fetch Messages for Active Chat
  useEffect(() => {
    if (!user || !activeChatId) return;
    const msgRef = collection(db, 'artifacts', appId, 'users', user.uid, 'chats', activeChatId, 'messages');
    const unsubscribe = onSnapshot(msgRef, (snapshot) => {
      const msgData = snapshot.docs.map(doc => doc.data()).sort((a, b) => a.timestamp - b.timestamp);
      setMessages(msgData);
    });
    return () => unsubscribe();
  }, [user, activeChatId]);

  useEffect(() => {
    scrollRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages, isLoading]);

  const handleNewChat = async () => {
    if (!user) return;
    const chatRef = collection(db, 'artifacts', appId, 'users', user.uid, 'chats');
    const newChat = await addDoc(chatRef, { 
      title: 'שיחה חדשה', 
      createdAt: Date.now() 
    });
    setActiveChatId(newChat.id);
    setIsSidebarOpen(false);
  };

  const handleSend = async () => {
    if (!input.trim() || isLoading || !user || !activeChatId) return;
    const userMsgText = input.trim();
    setInput('');
    setIsLoading(true);

    const msgRef = collection(db, 'artifacts', appId, 'users', user.uid, 'chats', activeChatId, 'messages');
    
    // Save User Message
    await addDoc(msgRef, {
      role: 'user',
      content: userMsgText,
      timestamp: Date.now()
    });

    // Update Chat Title if it's the first message
    if (messages.length === 0) {
      const chatDoc = doc(db, 'artifacts', appId, 'users', user.uid, 'chats', activeChatId);
      await setDoc(chatDoc, { title: userMsgText.slice(0, 30) }, { merge: true });
    }

    try {
      const apiKey = ""; // Managed by environment
      const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          contents: [{ parts: [{ text: userMsgText }] }],
          systemInstruction: { parts: [{ text: SYSTEM_PROMPT }] }
        })
      });
      const data = await response.json();
      const aiReply = data.candidates?.[0]?.content?.parts?.[0]?.text || "מצטער אבא, הייתה שגיאה.";

      // Save AI Reply
      await addDoc(msgRef, {
        role: 'assistant',
        content: aiReply,
        timestamp: Date.now()
      });
    } catch (e) {
      console.error(e);
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <div className="flex h-screen bg-[#050505] text-gray-100 font-sans selection:bg-emerald-500/30 overflow-hidden" dir="rtl">
      
      <Sidebar 
        chats={chats} 
        activeChatId={activeChatId} 
        onSelectChat={setActiveChatId} 
        onNewChat={handleNewChat}
        isMobileOpen={isSidebarOpen}
        toggleMobile={() => setIsSidebarOpen(!isSidebarOpen)}
      />

      {/* Main Content */}
      <main className="flex-1 flex flex-col relative bg-[#050505]">
        
        {/* Header */}
        <header className="h-16 flex items-center justify-between px-6 border-b border-white/5 bg-black/40 backdrop-blur-xl z-20">
          <div className="flex items-center gap-4">
            <button onClick={() => setIsSidebarOpen(!isSidebarOpen)} className="md:hidden p-2 hover:bg-white/5 rounded-lg border border-white/10 transition-all">
              <Menu size={20} />
            </button>
            <div className="flex flex-col">
              <h2 className="font-bold text-sm tracking-wide">SkorChat <span className="text-emerald-500">Pro</span></h2>
              <p className="text-[9px] text-gray-500 font-bold uppercase tracking-widest">Eliav Skornick Edition</p>
            </div>
          </div>
          <div className="flex items-center gap-2">
            <div className="w-2 h-2 rounded-full bg-emerald-500 animate-pulse shadow-[0_0_8px_#10b981]"></div>
            <span className="text-[10px] text-gray-400 font-bold uppercase">Cloud Connected</span>
          </div>
        </header>

        {/* Message Window */}
        <div className="flex-1 overflow-y-auto custom-scrollbar p-6 space-y-8">
          <div className="max-w-3xl mx-auto space-y-6">
            {messages.map((msg, idx) => (
              <div key={idx} className={`flex gap-4 ${msg.role === 'user' ? 'flex-row-reverse' : ''} animate-in fade-in`}>
                <div className={`w-10 h-10 rounded-xl flex items-center justify-center flex-shrink-0 shadow-lg ${msg.role === 'user' ? 'bg-white text-black' : 'bg-emerald-500 text-black shadow-emerald-500/20'}`}>
                  {msg.role === 'user' ? <User size={20} /> : <Bot size={20} />}
                </div>
                <div className={`px-5 py-3 rounded-2xl text-[16px] leading-relaxed shadow-xl ${msg.role === 'user' ? 'bg-emerald-600 text-white rounded-tr-none' : 'bg-white/5 border border-white/10 text-gray-100 rounded-tl-none'}`}>
                  <p className="whitespace-pre-wrap">{msg.content}</p>
                </div>
              </div>
            ))}
            {isLoading && (
              <div className="flex gap-4 animate-pulse">
                <div className="w-10 h-10 rounded-xl bg-emerald-500/20 flex items-center justify-center text-emerald-400">
                  <Loader2 className="animate-spin" size={20} />
                </div>
                <div className="bg-white/5 border border-white/10 rounded-2xl px-6 py-3 text-sm text-gray-500 italic font-bold">
                  Skor Intelligence מעבד...
                </div>
              </div>
            )}
            <div ref={scrollRef} />
          </div>
        </div>

        {/* Input Bar */}
        <footer className="p-6 md:p-10 bg-gradient-to-t from-black via-black/90 to-transparent">
          <div className="max-w-3xl mx-auto relative group">
            <div className="absolute -inset-1 bg-gradient-to-r from-emerald-500 to-teal-500 rounded-[2rem] blur opacity-0 group-focus-within:opacity-10 transition-all duration-700"></div>
            <div className="relative flex items-end gap-3 bg-[#0d0d0d] border border-white/10 rounded-2xl p-3 focus-within:border-emerald-500/40 transition-all duration-300 shadow-2xl">
              <textarea 
                rows="1" 
                value={input}
                onChange={e => {
                  setInput(e.target.value);
                  e.target.style.height = 'auto';
                  e.target.style.height = Math.min(e.target.scrollHeight, 250) + 'px';
                }}
                onKeyDown={e => e.key === 'Enter' && !e.shiftKey && (e.preventDefault(), handleSend())}
                placeholder="איך SkorChat Pro יכול לשרת אותך היום?"
                className="flex-1 bg-transparent border-none focus:ring-0 py-3 px-4 text-white placeholder:text-gray-600 resize-none custom-scrollbar text-[16px]"
              ></textarea>
              <button 
                onClick={handleSend} 
                disabled={!input.trim() || isLoading} 
                className={`p-4 rounded-xl transition-all duration-300 ${input.trim() && !isLoading ? 'bg-emerald-500 text-black hover:scale-105 active:scale-95 shadow-[0_0_20px_rgba(16,185,129,0.4)]' : 'bg-white/5 text-gray-600 cursor-not-allowed'}`}
              >
                <Send size={24} />
              </button>
            </div>
            <div className="flex justify-between mt-4 px-6 text-[10px] text-gray-600 font-black uppercase tracking-[0.15em]">
              <p>Inventor: Eliav Skornick</p>
              <p>© Skor AI v7.0</p>
            </div>
          </div>
        </footer>
      </main>

      <style dangerouslySetInnerHTML={{ __html: `
        .custom-scrollbar::-webkit-scrollbar { width: 4px; }
        .custom-scrollbar::-webkit-scrollbar-track { background: transparent; }
        .custom-scrollbar::-webkit-scrollbar-thumb { background: #222; border-radius: 10px; }
        .custom-scrollbar::-webkit-scrollbar-thumb:hover { background: #333; }
        @keyframes fadeIn { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }
        .animate-in { animation: fadeIn 0.4s ease-out forwards; }
      `}} />
    </div>
  );
};

export default App;


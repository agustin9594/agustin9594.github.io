import React, { useState, useEffect, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { 
  getAuth, 
  signInAnonymously, 
  onAuthStateChanged,
  signInWithCustomToken
} from 'firebase/auth';
import { 
  getFirestore, 
  collection, 
  addDoc, 
  query, 
  orderBy, 
  onSnapshot,
  serverTimestamp
} from 'firebase/firestore';
import { 
  Send, 
  Mic, 
  Download, 
  MoreVertical, 
  StopCircle,
  ChevronLeft,
  Share
} from 'lucide-react';

// --- Configuración de Firebase (Idéntica) ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';

// --- Estilos iOS Específicos ---
// Evita el zoom en inputs en iPhone (font-size >= 16px) y maneja safe-areas
const IOS_SAFE_AREA_TOP = "env(safe-area-inset-top)";
const IOS_SAFE_AREA_BOTTOM = "env(safe-area-inset-bottom)";

export default function App() {
  const [user, setUser] = useState(null);
  const [messages, setMessages] = useState([]);
  const [transactions, setTransactions] = useState([]);
  const [inputText, setInputText] = useState("");
  const [isRecording, setIsRecording] = useState(false);
  const [loadingAI, setLoadingAI] = useState(false);
  const [showInstallHelp, setShowInstallHelp] = useState(false);
  
  const messagesEndRef = useRef(null);
  const recognitionRef = useRef(null);

  // Detectar si es iOS para mostrar ayuda
  useEffect(() => {
    const isIOS = /iPad|iPhone|iPod/.test(navigator.userAgent) && !window.MSStream;
    // Si es iOS y no está en modo standalone (pantalla completa), sugerir instalar
    const isStandalone = window.navigator.standalone || window.matchMedia('(display-mode: standalone)').matches;
    
    if (isIOS && !isStandalone) {
      // Mostrar popup de ayuda tras 2 segundos
      setTimeout(() => setShowInstallHelp(true), 2000);
    }
  }, []);

  // Auth y Datos (Igual que versión anterior)
  useEffect(() => {
    const initAuth = async () => {
      if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
        await signInWithCustomToken(auth, __initial_auth_token);
      } else {
        await signInAnonymously(auth);
      }
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, setUser);
    return () => unsubscribe();
  }, []);

  useEffect(() => {
    if (!user) return;
    const q = query(
      collection(db, 'artifacts', appId, 'users', user.uid, 'transactions'), 
      orderBy('date', 'desc')
    );
    const unsubscribe = onSnapshot(q, (snapshot) => {
      setTransactions(snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() })));
    });
    return () => unsubscribe();
  }, [user]);

  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [messages, loadingAI]);

  useEffect(() => {
    if (user && messages.length === 0) {
      setMessages([{
        id: 'welcome',
        role: 'system',
        text: '¡Hola! Soy tu contador personal IA. Toca el micrófono para registrar gastos.',
        timestamp: new Date()
      }]);
    }
  }, [user]);

  // --- Lógica IA (Optimizada) ---
  const processWithGemini = async (text) => {
    setLoadingAI(true);
    const apiKey = ""; 
    
    const recentTransactions = transactions.slice(0, 20).map(t => 
      `- ${t.type}: $${t.amount} (${t.category}/${t.subcategory})`
    ).join("\n");

    const systemPrompt = `
      Actúa como una app de finanzas.
      El usuario te hablará natural.
      
      CATEGORIAS:
      - Comida (cocinar, calle)
      - Alquiler (expensas, servicios, base)
      - Viaje (sube, uber)
      - Limpieza, Higiene, Educacion, Servicios, Emergencias.
      - Ingresos (rappi, guardias, ventas)

      SI ES REGISTRO (Gaste, Compre, Gané):
      Responde JSON: { "action": "add", "type": "gasto"|"ingreso", "amount": number, "category": string, "subcategory": string, "detail": string, "response_text": string }

      SI ES CONSULTA (Cuanto gaste, dame lista):
      Responde JSON: { "action": "query", "response_text": string }
      
      Usa estos datos recientes si preguntan balance:
      ${recentTransactions}
    `;

    try {
      const response = await fetch(
        `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`,
        {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            contents: [{ parts: [{ text: text }] }],
            systemInstruction: { parts: [{ text: systemPrompt }] },
            generationConfig: { responseMimeType: "application/json" }
          })
        }
      );

      const data = await response.json();
      const result = JSON.parse(data.candidates?.[0]?.content?.parts?.[0]?.text);
      
      if (result.action === "add") {
        await addDoc(collection(db, 'artifacts', appId, 'users', user.uid, 'transactions'), {
          ...result,
          date: serverTimestamp()
        });
        addMessage("system", result.response_text || "Anotado.");
      } else {
        // Calcular balance rápido si es consulta
        if (text.toLowerCase().includes('balance') || text.toLowerCase().includes('queda')) {
             const ing = transactions.filter(t => t.type === 'ingreso').reduce((a,b)=>a+b.amount,0);
             const gas = transactions.filter(t => t.type === 'gasto').reduce((a,b)=>a+b.amount,0);
             addMessage("system", `Balance actual:\nIngresos: $${ing}\nGastos: $${gas}\nTotal: $${ing - gas}`);
        } else {
             addMessage("system", result.response_text);
        }
      }

    } catch (e) {
      console.error(e);
      addMessage("system", "Error de conexión. Intenta de nuevo.");
    } finally {
      setLoadingAI(false);
    }
  };

  const addMessage = (role, text) => {
    setMessages(p => [...p, { id: Date.now(), role, text, timestamp: new Date() }]);
  };

  const handleSend = () => {
    if (!inputText.trim()) return;
    addMessage("user", inputText);
    processWithGemini(inputText);
    setInputText("");
  };

  // --- Voz (Safari Mobile Support) ---
  const startRecording = () => {
    const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
    if (!SpeechRecognition) {
      alert("Navegador no soportado. Usa Safari.");
      return;
    }
    const recognition = new SpeechRecognition();
    recognition.lang = 'es-AR';
    recognition.onstart = () => setIsRecording(true);
    recognition.onresult = (e) => setInputText(e.results[0][0].transcript);
    recognition.onend = () => setIsRecording(false);
    recognitionRef.current = recognition;
    recognition.start();
  };

  const stopRecording = () => recognitionRef.current?.stop();

  // --- Renderizado ---
  return (
    <div className="flex flex-col h-screen bg-[#f2f2f7] font-sans text-gray-900 overflow-hidden">
      
      {/* Header estilo iOS */}
      <div 
        className="bg-white/90 backdrop-blur-md border-b border-gray-200 px-4 flex items-center justify-between sticky top-0 z-20"
        style={{ paddingTop: `max(1rem, ${IOS_SAFE_AREA_TOP})`, paddingBottom: '0.8rem' }}
      >
        <div className="flex items-center gap-2 text-[#007AFF]">
          <ChevronLeft size={24} />
          <span className="text-lg">Atrás</span>
        </div>
        <div className="font-semibold text-black">Finanzas IA</div>
        <div className="flex gap-3 text-[#007AFF]">
            <Download size={22} onClick={() => alert("Función de descarga disponible")} />
            <MoreVertical size={22} />
        </div>
      </div>

      {/* Área de Chat */}
      <div className="flex-1 overflow-y-auto p-4 space-y-4 bg-[#e5e5ea]">
        {messages.map(msg => (
          <div key={msg.id} className={`flex ${msg.role === 'user' ? 'justify-end' : 'justify-start'}`}>
            <div className={`max-w-[75%] p-3 rounded-2xl text-[17px] shadow-sm ${
              msg.role === 'user' 
                ? 'bg-[#007AFF] text-white rounded-br-none' 
                : 'bg-white text-black rounded-bl-none'
            }`}>
              {msg.text}
            </div>
          </div>
        ))}
        {loadingAI && <div className="text-gray-400 text-xs text-center">Analizando...</div>}
        <div ref={messagesEndRef} />
      </div>

      {/* Input estilo iOS */}
      <div 
        className="bg-[#f9f9f9] border-t border-gray-200 p-2 flex items-end gap-2"
        style={{ paddingBottom: `max(0.5rem, ${IOS_SAFE_AREA_BOTTOM})` }}
      >
        <button className="p-2 text-[#007AFF]">
            <span className="text-2xl">+</span>
        </button>
        <div className="flex-1 bg-white border border-gray-300 rounded-full px-4 py-2 mb-1 flex items-center">
            <input 
                value={inputText}
                onChange={(e) => setInputText(e.target.value)}
                placeholder="Escribe o di tu gasto..."
                className="flex-1 bg-transparent outline-none text-[16px]" // 16px previene zoom
                onKeyDown={(e) => e.key === 'Enter' && handleSend()}
            />
        </div>
        {inputText ? (
            <button onClick={handleSend} className="p-3 bg-[#007AFF] rounded-full text-white mb-1">
                <Send size={18} />
            </button>
        ) : (
            <button 
                onTouchStart={startRecording} 
                onTouchEnd={stopRecording}
                onMouseDown={startRecording}
                onMouseUp={stopRecording}
                className={`p-3 rounded-full text-white mb-1 transition-all ${isRecording ? 'bg-red-500 scale-110' : 'bg-[#007AFF]'}`}
            >
                {isRecording ? <StopCircle size={18} /> : <Mic size={18} />}
            </button>
        )}
      </div>

      {/* Modal de Ayuda para Instalar (Solo visible si no está instalado) */}
      {showInstallHelp && (
        <div className="absolute inset-x-0 bottom-0 bg-white/95 backdrop-blur shadow-2xl rounded-t-2xl p-6 z-50 border-t border-gray-200 animate-slide-up pb-10">
            <div className="flex justify-between items-start mb-4">
                <h3 className="font-bold text-lg">Instalar App en iPhone</h3>
                <button onClick={() => setShowInstallHelp(false)} className="text-gray-400">✕</button>
            </div>
            <p className="text-sm text-gray-600 mb-4">
                Para tener esta app en tu pantalla de inicio y usarla a pantalla completa:
            </p>
            <ol className="text-sm space-y-3">
                <li className="flex items-center gap-3">
                    <span className="bg-gray-100 w-6 h-6 rounded-full flex items-center justify-center font-bold text-xs">1</span>
                    <span>Toca el botón <Share size={14} className="inline mx-1"/> <strong>Compartir</strong> abajo.</span>
                </li>
                <li className="flex items-center gap-3">
                    <span className="bg-gray-100 w-6 h-6 rounded-full flex items-center justify-center font-bold text-xs">2</span>
                    <span>Selecciona <strong>"Agregar a inicio"</strong> (Add to Home Screen).</span>
                </li>
            </ol>
            <div className="mt-6 flex justify-center">
                <div className="animate-bounce text-[#007AFF]">
                    ⬇
                </div>
            </div>
        </div>
      )}
    </div>
  );
}

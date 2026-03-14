<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>MarketMatch - Consultoría Inteligente</title>
    <!-- Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- React & ReactDOM -->
    <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <!-- Babel para JSX -->
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <!-- Lucide Icons -->
    <script src="https://unpkg.com/lucide@latest"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;700;900&display=swap');
        body { font-family: 'Inter', sans-serif; background-color: #050505; color: white; }
        .scrollbar-hide::-webkit-scrollbar { display: none; }
    </style>
</head>
<body>
    <div id="root"></div>

    <script type="text/babel">
        const { useState, useEffect, useRef } = React;
        
        // --- CONFIGURACIÓN ---
        const apiKey = ""; // La API Key se gestiona internamente en el entorno de ejecución

        const DATA_GEOGRAPHY = {
            "América": {
                "México": ["CDMX", "Monterrey", "Guadalajara", "Puebla", "Cancún"],
                "Colombia": ["Bogotá", "Medellín", "Cali", "Barranquilla", "Cartagena"],
                "Argentina": ["Buenos Aires", "Córdoba", "Rosario", "Mendoza", "La Plata"]
            },
            "Europa": {
                "España": ["Madrid", "Barcelona", "Valencia", "Sevilla", "Bilbao"],
                "Francia": ["París", "Lyon", "Marsella"]
            }
        };

        const CATEGORIES = [
            { id: 'f1', name: 'Fórmula 1' },
            { id: 'sports', name: 'Deportes' },
            { id: 'fashion', name: 'Moda y Ropa' },
            { id: 'tech', name: 'Tecnología' },
            { id: 'crypto', name: 'Cripto & Web3' }
        ];

        const MALE_NAMES = ["Carlos", "Juan", "Diego", "Andrés", "Ricardo", "Fernando"];
        const FEMALE_NAMES = ["Mariana", "Elena", "Lucía", "Sofía", "Valeria", "Isabel"];
        const LAST_NAMES = ["García", "Rodríguez", "López", "Martínez", "Pérez"];

        const generateProfiles = (category, isJunior, count = 20) => {
            return Array.from({ length: count }, (_, i) => {
                const isFemale = Math.random() > 0.5;
                const firstName = isFemale ? FEMALE_NAMES[Math.floor(Math.random() * FEMALE_NAMES.length)] : MALE_NAMES[Math.floor(Math.random() * MALE_NAMES.length)];
                const lastName = LAST_NAMES[Math.floor(Math.random() * LAST_NAMES.length)];
                const rating = (isJunior ? 3.5 + Math.random() * 1.5 : 4.5 + Math.random() * 0.5).toFixed(1);
                const genderSlug = isFemale ? 'women' : 'men';
                const photoId = Math.floor(Math.random() * 95);

                return {
                    id: Math.random(),
                    name: `${firstName} ${lastName}`,
                    gender: isFemale ? 'female' : 'male',
                    rating: parseFloat(rating),
                    experience: isJunior ? `${Math.floor(Math.random() * 11) + 1} meses` : `${Math.floor(Math.random() * 15) + 5} años`,
                    photo: `https://randomuser.me/api/portraits/${genderSlug}/${photoId}.jpg`,
                    isJunior
                };
            });
        };

        const App = () => {
            const [step, setStep] = useState('search');
            const [filters, setFilters] = useState({ continent: '', country: '', city: '', category: '' });
            const [results, setResults] = useState({ seniors: [], juniors: [] });
            const [selectedSpecialist, setSelectedSpecialist] = useState(null);
            const [chatMessages, setChatMessages] = useState([]);
            const [inputValue, setInputValue] = useState('');
            const [isTyping, setIsTyping] = useState(false);
            const chatEndRef = useRef(null);

            useEffect(() => { chatEndRef.current?.scrollIntoView({ behavior: 'smooth' }); }, [chatMessages]);

            const handleSearch = () => {
                if (filters.country && filters.city && filters.category) {
                    setStep('results');
                    setResults({ 
                        seniors: generateProfiles(filters.category, false, 20), 
                        juniors: generateProfiles(filters.category, true, 20) 
                    });
                }
            };

            const openChat = (spec) => {
                setSelectedSpecialist(spec);
                setChatMessages([{ role: 'ai', text: `Hola, soy experto en ${CATEGORIES.find(c => c.id === filters.category).name}. ¿En qué puedo ayudarte hoy?` }]);
                setStep('chat');
            };

            const sendMessage = async () => {
                if (!inputValue.trim()) return;
                const userMsg = { role: 'user', text: inputValue };
                setChatMessages(prev => [...prev, userMsg]);
                setInputValue('');
                setIsTyping(true);

                try {
                    const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
                        method: 'POST',
                        headers: { 'Content-Type': 'application/json' },
                        body: JSON.stringify({
                            contents: [{ parts: [{ text: `El cliente dice: "${inputValue}". Responde como experto en ${filters.category}, sé breve, no digas tu nombre y termina con una pregunta de calificación del proyecto.` }] }],
                            systemInstruction: { parts: [{ text: `Eres un consultor experto. Reglas: 1. No digas tu nombre. 2. Máximo 2 líneas. 3. Termina siempre con pregunta.` }] }
                        })
                    });
                    const data = await response.json();
                    const text = data.candidates?.[0]?.content?.parts?.[0]?.text || "¿Cuál es el presupuesto de tu proyecto?";
                    setChatMessages(prev => [...prev, { role: 'ai', text }]);
                } catch (e) {
                    setChatMessages(prev => [...prev, { role: 'ai', text: "Entiendo. ¿Para cuándo necesitas tener este proyecto terminado?" }]);
                } finally {
                    setIsTyping(false);
                }
            };

            return (
                <div className="min-h-screen flex flex-col">
                    <header className="p-6 border-b border-white/5 flex justify-between items-center bg-black">
                        <div className="flex items-center gap-2 cursor-pointer" onClick={() => setStep('search')}>
                            <div className="bg-amber-500 w-8 h-8 rounded-lg flex items-center justify-center font-black text-black">M</div>
                            <h1 className="text-xl font-black uppercase tracking-tighter">Market<span className="text-amber-500">Match</span></h1>
                        </div>
                    </header>

                    <main className="flex-1 max-w-4xl mx-auto w-full p-6">
                        {step === 'search' && (
                            <div className="py-10 space-y-8 text-center">
                                <h2 className="text-5xl font-black uppercase tracking-tighter">Talento <span className="text-amber-500">Global</span></h2>
                                <div className="bg-white/5 p-8 rounded-[2rem] border border-white/10 grid gap-4 text-left">
                                    <select className="bg-black border border-white/10 p-4 rounded-xl outline-none" onChange={(e) => setFilters({...filters, continent: e.target.value})}>
                                        <option value="">Continente...</option>
                                        {Object.keys(DATA_GEOGRAPHY).map(c => <option key={c} value={c}>{c}</option>)}
                                    </select>
                                    <select className="bg-black border border-white/10 p-4 rounded-xl outline-none" onChange={(e) => setFilters({...filters, country: e.target.value})}>
                                        <option value="">País...</option>
                                        {filters.continent && Object.keys(DATA_GEOGRAPHY[filters.continent]).map(c => <option key={c} value={c}>{c}</option>)}
                                    </select>
                                    <select className="bg-black border border-white/10 p-4 rounded-xl outline-none" onChange={(e) => setFilters({...filters, city: e.target.value})}>
                                        <option value="">Ciudad...</option>
                                        {filters.country && DATA_GEOGRAPHY[filters.continent][filters.country].map(c => <option key={c} value={c}>{c}</option>)}
                                    </select>
                                    <select className="bg-black border border-white/10 p-4 rounded-xl outline-none" onChange={(e) => setFilters({...filters, category: e.target.value})}>
                                        <option value="">Especialidad...</option>
                                        {CATEGORIES.map(c => <option key={c.id} value={c.id}>{c.name}</option>)}
                                    </select>
                                    <button onClick={handleSearch} className="bg-amber-500 text-black font-black p-4 rounded-xl uppercase tracking-widest mt-4">Escanear Ciudad</button>
                                </div>
                            </div>
                        )}

                        {step === 'results' && (
                            <div className="space-y-8 animate-in fade-in duration-500">
                                <div className="flex items-center justify-between">
                                    <h2 className="text-2xl font-black uppercase">Resultados en {filters.city}</h2>
                                    <button onClick={() => setStep('search')} className="text-xs text-white/40 uppercase">Atrás</button>
                                </div>
                                <div className="grid gap-4">
                                    {results.seniors.map(s => (
                                        <div key={s.id} className="bg-white/5 p-4 rounded-2xl flex items-center justify-between border border-white/5">
                                            <div className="flex items-center gap-4">
                                                <img src={s.photo} className="w-12 h-12 rounded-lg grayscale" />
                                                <div>
                                                    <p className="font-bold text-sm">{s.name}</p>
                                                    <p className="text-[10px] text-amber-500 font-bold uppercase">{s.experience} Exp</p>
                                                </div>
                                            </div>
                                            <button onClick={() => openChat(s)} className="bg-amber-500 text-black px-4 py-2 rounded-lg text-[10px] font-black uppercase">Chatear</button>
                                        </div>
                                    ))}
                                </div>
                            </div>
                        )}

                        {step === 'chat' && (
                            <div className="h-[600px] border border-white/10 rounded-3xl flex flex-col overflow-hidden bg-black/50">
                                <div className="p-4 border-b border-white/5 flex items-center gap-4">
                                    <button onClick={() => setStep('results')} className="p-2"><i data-lucide="chevron-left"></i></button>
                                    <img src={selectedSpecialist.photo} className="w-10 h-10 rounded-lg" />
                                    <p className="font-black text-sm">{selectedSpecialist.name}</p>
                                </div>
                                <div className="flex-1 overflow-y-auto p-6 space-y-4 scrollbar-hide">
                                    {chatMessages.map((m, i) => (
                                        <div key={i} className={`flex ${m.role === 'user' ? 'justify-end' : 'justify-start'}`}>
                                            <div className={`max-w-[80%] p-4 rounded-2xl text-sm ${m.role === 'user' ? 'bg-amber-500 text-black font-bold' : 'bg-white/10'}`}>
                                                {m.text}
                                            </div>
                                        </div>
                                    ))}
                                    {isTyping && <div className="text-xs text-amber-500 animate-pulse">Escribiendo...</div>}
                                    <div ref={chatEndRef} />
                                </div>
                                <div className="p-4 border-t border-white/5 flex gap-2">
                                    <input 
                                        type="text" 
                                        className="flex-1 bg-white/5 border border-white/10 rounded-xl px-4 text-sm outline-none" 
                                        placeholder="Escribe algo..."
                                        value={inputValue}
                                        onChange={(e) => setInputValue(e.target.value)}
                                        onKeyPress={(e) => e.key === 'Enter' && sendMessage()}
                                    />
                                    <button onClick={sendMessage} className="bg-amber-500 text-black p-3 rounded-xl"><i data-lucide="send"></i></button>
                                </div>
                            </div>
                        )}
                    </main>
                </div>
            );
        };

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
        
        // Iniciar iconos después de que React cargue
        setTimeout(() => lucide.createIcons(), 500);
    </script>
</body>
</html>

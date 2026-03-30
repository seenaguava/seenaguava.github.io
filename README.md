[index.css](https://github.com/user-attachments/files/26348198/index.css)# seenaguava.github.io
seenaguava
[index.html](https://github.com/user-attachments/files/26348170/index.html)
[metadata.json](https://github.com/user-attachments/files/26348171/metadata.json)[README.md](https://github.com/user-attachments/files/26348175/README.md)
[package.json](https://github.com/user-attachments/files/26348174/package.json)
[package-lock.json](https://github.com/user-attachments/files/26348172/package-lock.json)
[tsconfig.json](https://github.com/user-attachments/files/26348184/tsconfig.json)
[vite.config.ts](https://github.com/user-attachments/files/26348188/vite.config.ts)[App.tsx](https://github.com/user-attachments/files/26348196/App.tsx)
/**
 * @license
 * SPDX-License-Identifier: Apache-2.0
 */
[Uploading index.css[main.tsx](https://github.com/user-attachments/files/26348199/main.tsx)…]()
[RubiksCube.tsx](https://github.com/user-attachments/files/26348201/RubiksCube.tsx)import { useState, useRef, useEffect, useCallback } from 'react';
import { Canvas, useFrame } from '@react-three/fiber';
import { OrbitControls, PerspectiveCamera } from '@react-three/drei';
import * as THREE from 'three';
import { motion, AnimatePresence } from 'motion/react';
import { RotateCcw, Shuffle, MessageSquare, X, Send, HelpCircle, ChevronRight, ChevronLeft } from 'lucide-react';
import { GoogleGenAI } from "@google/genai";
import ReactMarkdown from 'react-markdown';
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

/**
 * Utility for Tailwind classes
 */
function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}

// --- Types ---

type Face = 'U' | 'D' | 'L' | 'R' | 'F' | 'B';
type Axis = 'x' | 'y' | 'z';

interface CubieState {
  id: string;
  position: [number, number, number]; // Initial grid position [-1, 0, 1]
  rotation: THREE.Euler;
  colors: {
    top: string;
    bottom: string;
    left: string;
    right: string;
    front: string;
    back: string;
  };
}

// --- Constants ---

const COLORS = {
  white: '#FFFFFF',
  yellow: '#FFD500',
  orange: '#FF5800',
  red: '#C41E3A',
  green: '#009E60',
  blue: '#0051BA',
  black: '#111111',
};

const INITIAL_CUBIES: CubieState[] = [];
for (let x = -1; x <= 1; x++) {
  for (let y = -1; y <= 1; y++) {
    for (let z = -1; z <= 1; z++) {
      if (x === 0 && y === 0 && z === 0) continue; // Core is empty

      INITIAL_CUBIES.push({
        id: `cubie-${x}-${y}-${z}`,
        position: [x, y, z],
        rotation: new THREE.Euler(0, 0, 0),
        colors: {
          top: y === 1 ? COLORS.white : COLORS.black,
          bottom: y === -1 ? COLORS.yellow : COLORS.black,
          left: x === -1 ? COLORS.orange : COLORS.black,
          right: x === 1 ? COLORS.red : COLORS.black,
          front: z === 1 ? COLORS.green : COLORS.black,
          back: z === -1 ? COLORS.blue : COLORS.black,
        },
      });
    }
  }
}

// --- Components ---

const Cubie = ({ state }: { state: CubieState }) => {
  return (
    <mesh position={state.position} rotation={state.rotation}>
      <boxGeometry args={[0.95, 0.95, 0.95]} />
      <meshStandardMaterial attach="material-0" color={state.colors.right} roughness={0.1} metalness={0.1} />
      <meshStandardMaterial attach="material-1" color={state.colors.left} roughness={0.1} metalness={0.1} />
      <meshStandardMaterial attach="material-2" color={state.colors.top} roughness={0.1} metalness={0.1} />
      <meshStandardMaterial attach="material-3" color={state.colors.bottom} roughness={0.1} metalness={0.1} />
      <meshStandardMaterial attach="material-4" color={state.colors.front} roughness={0.1} metalness={0.1} />
      <meshStandardMaterial attach="material-5" color={state.colors.back} roughness={0.1} metalness={0.1} />
    </mesh>
  );
};

export default function RubiksCubeApp() {
  const [cubies, setCubies] = useState<CubieState[]>(INITIAL_CUBIES);
  const [isRotating, setIsRotating] = useState(false);
  const [chatOpen, setChatOpen] = useState(false);
  
  // Timer State
  const [timerRunning, setTimerRunning] = useState(false);
  const [startTime, setStartTime] = useState<number | null>(null);
  const [elapsedTime, setElapsedTime] = useState(0);
  const [bestTime, setBestTime] = useState<number | null>(null);
  const [isSolved, setIsSolved] = useState(true);

  const [messages, setMessages] = useState<{ role: 'user' | 'model'; text: string }[]>([
    { role: 'model', text: "Hi! I'm your Rubik's Cube assistant. Need help solving the cube or want to learn some algorithms?" }
  ]);
  const [input, setInput] = useState('');
  const [loading, setLoading] = useState(false);

  const groupRef = useRef<THREE.Group>(null);

  // Timer Logic
  useEffect(() => {
    let interval: NodeJS.Timeout;
    if (timerRunning && startTime) {
      interval = setInterval(() => {
        setElapsedTime(Date.now() - startTime);
      }, 10);
    }
    return () => clearInterval(interval);
  }, [timerRunning, startTime]);

  const checkIsSolved = (currentCubies: CubieState[]) => {
    return currentCubies.every(cubie => {
      const [ix, iy, iz] = cubie.id.split('-').slice(1).map(Number);
      const [cx, cy, cz] = cubie.position;
      
      // Position must match
      if (ix !== cx || iy !== cy || iz !== cz) return false;
      
      // Rotation must be effectively identity (multiple of 2PI)
      const q = new THREE.Quaternion().setFromEuler(cubie.rotation);
      const identity = new THREE.Quaternion();
      const angle = q.angleTo(identity);
      
      // Allow for small floating point errors, but must be close to 0 or 2PI
      return angle < 0.1 || angle > Math.PI * 1.9;
    });
  };

  const rotateFace = useCallback((face: Face, clockwise: boolean = true) => {
    if (isRotating) return;
    setIsRotating(true);

    const axis: Axis = (face === 'L' || face === 'R') ? 'x' : (face === 'U' || face === 'D') ? 'y' : 'z';
    const actualLayer = (face === 'R' || face === 'U' || face === 'F') ? 1 : -1;
    const angle = (clockwise ? -Math.PI / 2 : Math.PI / 2) * (face === 'L' || face === 'D' || face === 'B' ? -1 : 1);

    const rotationMatrix = new THREE.Matrix4();
    if (axis === 'x') rotationMatrix.makeRotationX(angle);
    else if (axis === 'y') rotationMatrix.makeRotationY(angle);
    else rotationMatrix.makeRotationZ(angle);

    setCubies(prev => {
      const next = prev.map(cubie => {
        const pos = cubie.position;
        const match = axis === 'x' ? pos[0] === actualLayer : axis === 'y' ? pos[1] === actualLayer : pos[2] === actualLayer;

        if (match) {
          const vec = new THREE.Vector3(...pos);
          vec.applyMatrix4(rotationMatrix);
          
          const currentQuaternion = new THREE.Quaternion().setFromEuler(cubie.rotation);
          const rotationQuaternion = new THREE.Quaternion().setFromRotationMatrix(rotationMatrix);
          currentQuaternion.premultiply(rotationQuaternion);
          
          return {
            ...cubie,
            position: [Math.round(vec.x), Math.round(vec.y), Math.round(vec.z)] as [number, number, number],
            rotation: new THREE.Euler().setFromQuaternion(currentQuaternion),
          };
        }
        return cubie;
      });

      // Check for solve
      if (timerRunning) {
        const solved = checkIsSolved(next);
        if (solved) {
          setTimerRunning(false);
          setIsSolved(true);
          if (!bestTime || elapsedTime < bestTime) {
            setBestTime(elapsedTime);
          }
        }
      } else {
        setIsSolved(checkIsSolved(next));
      }

      return next;
    });

    setTimeout(() => setIsRotating(false), 300);
  }, [isRotating, timerRunning, elapsedTime, bestTime]);

  const startSolve = async () => {
    reset();
    setIsSolved(false);
    
    // Scramble
    const moves: Face[] = ['U', 'D', 'L', 'R', 'F', 'B'];
    for (let i = 0; i < 15; i++) {
      const move = moves[Math.floor(Math.random() * moves.length)];
      rotateFace(move, Math.random() > 0.5);
      await new Promise(r => setTimeout(r, 160));
    }

    setStartTime(Date.now());
    setElapsedTime(0);
    setTimerRunning(true);
  };

  const reset = () => {
    setCubies(INITIAL_CUBIES);
    setTimerRunning(false);
    setElapsedTime(0);
    setIsSolved(true);
  };

  const handleSend = async () => {
    if (!input.trim() || loading) return;
    
    const userMsg = input;
    setInput('');
    setMessages(prev => [...prev, { role: 'user', text: userMsg }]);
    setLoading(true);

    try {
      const ai = new GoogleGenAI({ apiKey: process.env.GEMINI_API_KEY! });
      const model = ai.models.generateContent({
        model: "gemini-3-flash-preview",
        contents: [
          { role: 'user', parts: [{ text: `You are a Rubik's Cube expert. Help the user solve their cube. User says: ${userMsg}` }] }
        ],
        config: {
          systemInstruction: "You are a friendly Rubik's Cube expert. Provide clear, step-by-step advice on solving the cube, explain notation (U, D, L, R, F, B), and common algorithms like the Sexy Move or Sune. Keep responses concise and helpful."
        }
      });

      const response = await model;
      setMessages(prev => [...prev, { role: 'model', text: response.text || "I'm sorry, I couldn't process that." }]);
    } catch (error) {
      console.error(error);
      setMessages(prev => [...prev, { role: 'model', text: "Sorry, I had trouble connecting to my brain. Please try again." }]);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="relative w-full h-screen bg-[#0a0a0a] overflow-hidden font-sans text-white">
      {/* 3D Canvas */}
      <div className="absolute inset-0 z-0">
        <Canvas shadows dpr={[1, 2]}>
          <PerspectiveCamera makeDefault position={[5, 5, 5]} fov={50} />
          <ambientLight intensity={1.5} />
          <pointLight position={[10, 10, 10]} intensity={2} />
          <spotLight position={[-10, 10, 10]} angle={0.15} penumbra={1} intensity={2} castShadow />
          <directionalLight position={[0, 10, 0]} intensity={1} />
          
          <group ref={groupRef}>
            {cubies.map(cubie => (
              <Cubie key={cubie.id} state={cubie} />
            ))}
          </group>

          <OrbitControls 
            enablePan={false} 
            minDistance={4} 
            maxDistance={10} 
            makeDefault 
            enabled={!isRotating}
          />
        </Canvas>
      </div>

      {/* UI Overlay */}
      <div className="absolute inset-0 pointer-events-none flex flex-col justify-between p-6 md:p-10">
        {/* Header */}
        <div className="flex justify-between items-start pointer-events-auto">
          <div className="flex flex-col">
            <h1 className="text-4xl md:text-6xl font-black tracking-tighter uppercase italic text-white/90">
              Rubik's <span className="text-orange-500">Cube</span>
            </h1>
            <div className="flex items-center gap-4 mt-2">
              <div className="bg-black/60 backdrop-blur-md border border-white/10 px-4 py-2 rounded-xl">
                <p className="text-[10px] font-mono tracking-widest uppercase opacity-50 mb-1">Timer</p>
                <p className={cn(
                  "text-2xl font-mono font-bold tabular-nums",
                  timerRunning ? "text-orange-500" : "text-white"
                )}>
                  {(elapsedTime / 1000).toFixed(2)}s
                </p>
              </div>
              {bestTime && (
                <div className="bg-black/60 backdrop-blur-md border border-white/10 px-4 py-2 rounded-xl">
                  <p className="text-[10px] font-mono tracking-widest uppercase opacity-50 mb-1">Best</p>
                  <p className="text-2xl font-mono font-bold text-green-500">
                    {(bestTime / 1000).toFixed(2)}s
                  </p>
                </div>
              )}
            </div>
          </div>
          
          <button 
            onClick={() => setChatOpen(!chatOpen)}
            className="w-12 h-12 rounded-full bg-white/10 backdrop-blur-md border border-white/20 flex items-center justify-center hover:bg-white/20 transition-all active:scale-95"
          >
            {chatOpen ? <X size={20} /> : <MessageSquare size={20} />}
          </button>
        </div>

        {/* Controls */}
        <div className="flex flex-col md:flex-row items-end md:items-center justify-between gap-6 pointer-events-auto">
          {/* Face Rotation Buttons */}
          <div className="grid grid-cols-3 gap-2 bg-black/40 backdrop-blur-xl p-4 rounded-2xl border border-white/10">
            {(['U', 'D', 'L', 'R', 'F', 'B'] as Face[]).map(face => (
              <div key={face} className="flex gap-1">
                <button 
                  onClick={() => rotateFace(face, true)}
                  disabled={isRotating}
                  className="w-10 h-10 rounded-lg bg-white/5 hover:bg-white/20 border border-white/10 flex items-center justify-center font-bold text-sm transition-all disabled:opacity-50"
                >
                  {face}
                </button>
                <button 
                  onClick={() => rotateFace(face, false)}
                  disabled={isRotating}
                  className="w-10 h-10 rounded-lg bg-white/5 hover:bg-white/20 border border-white/10 flex items-center justify-center font-bold text-sm transition-all disabled:opacity-50"
                >
                  {face}'
                </button>
              </div>
            ))}
          </div>

          {/* Action Buttons */}
          <div className="flex gap-3">
            {!timerRunning ? (
              <button 
                onClick={startSolve}
                disabled={isRotating}
                className="flex items-center gap-2 px-8 py-4 rounded-full bg-orange-600 hover:bg-orange-500 text-white font-bold uppercase tracking-wider text-sm shadow-lg shadow-orange-900/20 transition-all active:scale-95 disabled:opacity-50"
              >
                <Shuffle size={18} />
                Start Timer
              </button>
            ) : (
              <button 
                onClick={() => setTimerRunning(false)}
                className="flex items-center gap-2 px-8 py-4 rounded-full bg-red-600 hover:bg-red-500 text-white font-bold uppercase tracking-wider text-sm transition-all active:scale-95"
              >
                <X size={18} />
                Stop Timer
              </button>
            )}
            <button 
              onClick={reset}
              disabled={isRotating}
              className="flex items-center gap-2 px-6 py-3 rounded-full bg-white/10 hover:bg-white/20 border border-white/10 text-white font-bold uppercase tracking-wider text-sm transition-all active:scale-95 disabled:opacity-50"
            >
              <RotateCcw size={18} />
              Reset
            </button>
          </div>
        </div>
      </div>

      {/* Chat Sidebar */}
      <AnimatePresence>
        {chatOpen && (
          <motion.div 
            initial={{ x: '100%' }}
            animate={{ x: 0 }}
            exit={{ x: '100%' }}
            transition={{ type: 'spring', damping: 25, stiffness: 200 }}
            className="absolute top-0 right-0 w-full md:w-96 h-full bg-[#111] border-l border-white/10 z-50 flex flex-col shadow-2xl"
          >
            <div className="p-6 border-bottom border-white/10 flex justify-between items-center bg-black/20">
              <div className="flex items-center gap-3">
                <div className="w-8 h-8 rounded-lg bg-orange-500 flex items-center justify-center">
                  <HelpCircle size={18} className="text-black" />
                </div>
                <h2 className="font-bold uppercase tracking-tight">Gemini Assistant</h2>
              </div>
              <button onClick={() => setChatOpen(false)} className="opacity-50 hover:opacity-100">
                <X size={20} />
              </button>
            </div>

            <div className="flex-1 overflow-y-auto p-6 space-y-4 scrollbar-hide">
              {messages.map((msg, i) => (
                <div key={i} className={cn(
                  "flex flex-col max-w-[85%]",
                  msg.role === 'user' ? "ml-auto items-end" : "mr-auto items-start"
                )}>
                  <div className={cn(
                    "p-3 rounded-2xl text-sm leading-relaxed",
                    msg.role === 'user' 
                      ? "bg-orange-600 text-white rounded-tr-none" 
                      : "bg-white/10 text-white/90 rounded-tl-none border border-white/5"
                  )}>
                    <ReactMarkdown>{msg.text}</ReactMarkdown>
                  </div>
                  <span className="text-[10px] uppercase opacity-30 mt-1 font-mono tracking-tighter">
                    {msg.role === 'user' ? 'You' : 'Assistant'}
                  </span>
                </div>
              ))}
              {loading && (
                <div className="flex gap-1 items-center opacity-50 ml-2">
                  <div className="w-1 h-1 bg-white rounded-full animate-bounce" />
                  <div className="w-1 h-1 bg-white rounded-full animate-bounce [animation-delay:0.2s]" />
                  <div className="w-1 h-1 bg-white rounded-full animate-bounce [animation-delay:0.4s]" />
                </div>
              )}
            </div>

            <div className="p-6 bg-black/40 border-t border-white/10">
              <div className="relative">
                <input 
                  type="text"
                  value={input}
                  onChange={(e) => setInput(e.target.value)}
                  onKeyDown={(e) => e.key === 'Enter' && handleSend()}
                  placeholder="Ask for help..."
                  className="w-full bg-white/5 border border-white/10 rounded-xl py-3 pl-4 pr-12 text-sm focus:outline-none focus:border-orange-500/50 transition-all"
                />
                <button 
                  onClick={handleSend}
                  disabled={loading || !input.trim()}
                  className="absolute right-2 top-1/2 -translate-y-1/2 w-8 h-8 rounded-lg bg-orange-600 flex items-center justify-center hover:bg-orange-500 transition-all disabled:opacity-50 disabled:grayscale"
                >
                  <Send size={14} />
                </button>
              </div>
              <p className="text-[10px] opacity-30 text-center mt-4 uppercase tracking-widest">
                Powered by Gemini AI
              </p>
            </div>
          </motion.div>
        )}
      </AnimatePresence>

      {/* Background Accents */}
      <div className="absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 w-[800px] h-[800px] bg-orange-500/5 blur-[120px] rounded-full pointer-events-none -z-10" />
      <div className="absolute bottom-0 left-0 w-full h-32 bg-gradient-to-t from-black to-transparent pointer-events-none z-0" />
    </div>
  );
}


import RubiksCubeApp from './components/RubiksCube';

export default function App() {
  return (
    <div className="min-h-screen bg-black">
      <RubiksCubeApp />
    </div>
  );
}

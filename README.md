# treadmill
Treadmill Calculator
import React, { useState, useEffect, useMemo } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInWithCustomToken, signInAnonymously, onAuthStateChanged, setPersistence, browserLocalPersistence } from 'firebase/auth';
import { getFirestore, collection, addDoc, onSnapshot, doc, deleteDoc, query, serverTimestamp } from 'firebase/firestore';

// --- Helper Functions & Constants ---
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';

const getFirebaseConfig = () => {
    try {
        return typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : null;
    } catch (e) {
        console.error("Error parsing Firebase config:", e);
        return null;
    }
};

// --- SVG Icons ---
const DumbbellIcon = () => (
    <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" className="h-8 w-8 mr-3 text-cyan-400">
        <path d="M14.4 14.4 9.6 9.6" /><path d="M18.657 21.485a2 2 0 1 1-2.829-2.828l-1.767 1.768a2 2 0 1 1-2.829-2.829l6.364-6.364a2 2 0 1 1 2.829 2.829l-1.768 1.767a2 2 0 1 1 2.828 2.829z" /><path d="m21.5 21.5-1.4-1.4" /><path d="M3.9 3.9 2.5 2.5" /><path d="M6.343 12.343a2 2 0 1 1-2.829-2.829l1.768-1.767a2 2 0 1 1 2.829-2.829l6.363 6.364a2 2 0 1 1-2.829 2.829l-1.767-1.768a2 2 0 1 1-2.829 2.829z" />
    </svg>
);

const TrashIcon = ({ className }) => (
    <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" className={className}>
        <path d="M3 6h18" /><path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2" /><line x1="10" y1="11" x2="10" y2="17" /><line x1="14" y1="11" x2="14" y2="17" />
    </svg>
);

const RunIcon = () => (
     <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" className="h-4 w-4 text-gray-400"><path d="M16 13a3 3 0 1 0-3-3 3 3 0 0 0 3 3z"/><path d="M13 13V2a3 3 0 0 0-6 0v4"/><path d="m11.1 13.7-2.5 2.5a4.4 4.4 0 0 0 0 6.2l.2.2a4.4 4.4 0 0 0 6.2 0l2-2"/><path d="m5.5 18 3-3"/></svg>
);

const ClockIcon = () => (
    <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" className="h-4 w-4 text-gray-400"><circle cx="12" cy="12" r="10"/><polyline points="12 6 12 12 16 14"/></svg>
);

const FlameIcon = () => (
    <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" className="h-4 w-4 text-gray-400"><path d="M8.5 14.5A2.5 2.5 0 0 0 11 12c0-1.38-.5-2-1-3-1.072-2.143-.224-4.054 2-6 .5 2.5 2 4.9 4 6.5 2 1.6 3 3.5 3 5.5a7 7 0 1 1-14 0c0-1.153.433-2.294 1-3a2.5 2.5 0 0 0 2.5 2.5z"/></svg>
);

const GaugeIcon = () => (
    <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" className="h-4 w-4 text-gray-400"><path d="m12 14 4-4"/><path d="M3.34 19a10 10 0 1 1 17.32 0"/></svg>
);


// --- Main App Component ---
export default function App() {
    // Firebase state
    const [db, setDb] = useState(null);
    const [auth, setAuth] = useState(null);
    const [userId, setUserId] = useState(null);
    const [isAuthReady, setIsAuthReady] = useState(false);

    // App state
    const [workouts, setWorkouts] = useState([]);
    const [distance, setDistance] = useState('');
    const [duration, setDuration] = useState('');
    const [calories, setCalories] = useState('');
    const [error, setError] = useState('');

    // Gemini API State
    const [insights, setInsights] = useState('');
    const [isGenerating, setIsGenerating] = useState(false);
    const [apiError, setApiError] = useState('');


    // --- Firebase Initialization and Auth ---
    useEffect(() => {
        const firebaseConfig = getFirebaseConfig();
        if (firebaseConfig) {
            const app = initializeApp(firebaseConfig);
            const firestoreDb = getFirestore(app);
            const firebaseAuth = getAuth(app);
            setDb(firestoreDb);
            setAuth(firebaseAuth);

            const unsubscribe = onAuthStateChanged(firebaseAuth, async (user) => {
                if (user) {
                    setUserId(user.uid);
                } else {
                    try {
                        const token = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
                        if (token) {
                            await signInWithCustomToken(firebaseAuth, token);
                        } else {
                            await signInAnonymously(firebaseAuth);
                        }
                    } catch (err) {
                        console.error("Authentication failed:", err);
                        setError("Could not connect to the workout service.");
                    }
                }
                setIsAuthReady(true);
            });

            return () => unsubscribe();
        } else {
            setError("Firebase configuration is missing.");
            setIsAuthReady(true);
        }
    }, []);
    
    // --- Firestore Data Fetching ---
    useEffect(() => {
        if (isAuthReady && db && userId) {
            const workoutsCollection = collection(db, `artifacts/${appId}/users/${userId}/workouts`);
            const q = query(workoutsCollection);

            const unsubscribe = onSnapshot(q, (querySnapshot) => {
                const workoutsData = [];
                querySnapshot.forEach((doc) => {
                    workoutsData.push({ id: doc.id, ...doc.data() });
                });
                // Sort by timestamp, newest first
                workoutsData.sort((a, b) => (b.createdAt?.seconds || 0) - (a.createdAt?.seconds || 0));
                setWorkouts(workoutsData);
            }, (err) => {
                console.error("Error fetching workouts:", err);
                setError("Failed to load workouts.");
            });

            return () => unsubscribe();
        }
    }, [db, userId, isAuthReady]);


    // --- Helper & Handler Functions ---
    const parseDurationToSeconds = (durationStr) => {
        const parts = durationStr.split(':').map(Number);
        if (parts.some(isNaN)) return null;
        let seconds = 0;
        if (parts.length === 3) { // HH:MM:SS
            seconds = parts[0] * 3600 + parts[1] * 60 + parts[2];
        } else if (parts.length === 2) { // MM:SS
            seconds = parts[0] * 60 + parts[1];
        } else if (parts.length === 1) { // SS
            seconds = parts[0];
        } else {
            return null;
        }
        return seconds;
    };

    const formatDuration = (totalSeconds) => {
        if (totalSeconds === null || totalSeconds === undefined || isNaN(totalSeconds)) {
            return '00:00:00';
        }
        const hours = Math.floor(totalSeconds / 3600);
        const minutes = Math.floor((totalSeconds % 3600) / 60);
        const seconds = Math.floor(totalSeconds % 60);
        return `${String(hours).padStart(2, '0')}:${String(minutes).padStart(2, '0')}:${String(seconds).padStart(2, '0')}`;
    };
    
    const calculatePace = (dist, durationStr) => {
        if (!dist || !durationStr || parseFloat(dist) <= 0) return '00:00';
        const totalSeconds = parseDurationToSeconds(durationStr);
        if (totalSeconds === null || totalSeconds <= 0) return '00:00';

        const paceInSeconds = totalSeconds / parseFloat(dist);
        const paceMinutes = Math.floor(paceInSeconds / 60);
        const paceSeconds = Math.floor(paceInSeconds % 60);

        return `${String(paceMinutes).padStart(2, '0')}:${String(paceSeconds).padStart(2, '0')}`;
    };
    
    const formatDate = (timestamp) => {
        if (!timestamp || !timestamp.seconds) {
            return new Date().toLocaleDateString();
        }
        return new Date(timestamp.seconds * 1000).toLocaleDateString('en-US', {
            year: 'numeric',
            month: 'long',
            day: 'numeric'
        });
    };

    const handleSubmit = async (e) => {
        e.preventDefault();
        if (!db || !userId) {
            setError("Database not connected.");
            return;
        }

        const distNum = parseFloat(distance);
        const calsNum = parseInt(calories, 10) || 0;
        const durationSec = parseDurationToSeconds(duration);

        if (!distNum || distNum <= 0) {
            setError("Please enter a valid distance.");
            return;
        }
        if (durationSec === null || durationSec <= 0) {
            setError("Please enter a valid duration (e.g., 46:51 or 00:46:51).");
            return;
        }
        if (calories && (isNaN(calsNum) || calsNum < 0)) {
             setError("Please enter a valid number for calories.");
             return;
        }
        setError('');

        try {
            const workoutsCollection = collection(db, `artifacts/${appId}/users/${userId}/workouts`);
            await addDoc(workoutsCollection, {
                distance: distNum,
                duration: durationSec,
                calories: calsNum,
                createdAt: serverTimestamp()
            });
            // Reset form
            setDistance('');
            setDuration('');
            setCalories('');
        } catch (err) {
            console.error("Error adding workout:", err);
            setError("Failed to save workout.");
        }
    };
    
    const handleDelete = async (id) => {
        if (!db || !userId) return;
        const workoutDocRef = doc(db, `artifacts/${appId}/users/${userId}/workouts`, id);
        try {
            await deleteDoc(workoutDocRef);
        } catch (err) {
            console.error("Error deleting workout:", err);
            setError("Failed to delete workout.");
        }
    };

    // --- Gemini API Call ---
    const getWorkoutInsights = async () => {
        if (workouts.length < 3) {
            setApiError("Log at least 3 workouts to get AI-powered insights.");
            return;
        }

        setIsGenerating(true);
        setInsights('');
        setApiError('');

        const systemPrompt = "You are a friendly and encouraging personal trainer. Analyze the user's recent treadmill workouts and provide a short, motivational summary of their progress. Then, suggest one specific, actionable idea for their next workout. Keep the entire response to 3-4 sentences. Format as plain text.";

        // Prepare last 10 workouts for context
        const recentWorkouts = workouts.slice(0, 10).map(w =>
            `Date: ${formatDate(w.createdAt)}, Distance: ${w.distance.toFixed(2)} mi, Duration: ${formatDuration(w.duration)}, Calories: ${w.calories} kcal`
        ).join('\n');

        const userQuery = `Here are my recent workouts:\n${recentWorkouts}`;

        const apiKey = ""; // Canvas will provide this
        const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-05-20:generateContent?key=${apiKey}`;

        const payload = {
            contents: [{ parts: [{ text: userQuery }] }],
            systemInstruction: {
                parts: [{ text: systemPrompt }]
            },
        };

        try {
            const response = await fetch(apiUrl, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(payload)
            });

            if (!response.ok) {
                throw new Error(`API call failed with status: ${response.status}`);
            }

            const result = await response.json();
            const text = result.candidates?.[0]?.content?.parts?.[0]?.text;

            if (text) {
                setInsights(text);
            } else {
                throw new Error("Could not parse AI response.");
            }
        } catch (err) {
            console.error("Gemini API error:", err);
            setApiError("Sorry, I couldn't generate insights right now. Please try again later.");
        } finally {
            setIsGenerating(false);
        }
    };

    // --- Memoized Calculations for Totals ---
    const totalStats = useMemo(() => {
        return workouts.reduce((acc, workout) => {
            acc.distance += workout.distance || 0;
            acc.calories += workout.calories || 0;
            acc.duration += workout.duration || 0;
            return acc;
        }, { distance: 0, calories: 0, duration: 0 });
    }, [workouts]);


    return (
        <div className="bg-gradient-to-br from-gray-900 to-black text-white min-h-screen font-sans p-4 sm:p-6 lg:p-8">
            <div className="max-w-4xl mx-auto">
                {/* Header */}
                <header className="flex items-center mb-8">
                    <DumbbellIcon/>
                    <h1 className="text-3xl sm:text-4xl font-bold text-transparent bg-clip-text bg-gradient-to-r from-gray-100 to-gray-400">Treadmill Pace Calculator</h1>
                </header>
                
                {error && (
                    <div className="bg-red-500/20 border border-red-500 text-red-300 px-4 py-3 rounded-lg mb-6 text-sm" role="alert">
                        <p>{error}</p>
                    </div>
                )}
                
                {/* Add Workout Form */}
                <div className="bg-white/5 backdrop-blur-lg p-6 rounded-2xl border border-white/10 shadow-lg mb-8">
                    <h2 className="text-xl font-semibold mb-4 text-gray-300">Log New Workout</h2>
                    <form onSubmit={handleSubmit} className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
                        {/* Distance */}
                        <div className="flex flex-col">
                            <label htmlFor="distance" className="text-sm font-medium text-gray-400 mb-1">Distance (mi)</label>
                            <input
                                id="distance"
                                type="number"
                                step="0.01"
                                value={distance}
                                onChange={(e) => setDistance(e.target.value)}
                                placeholder="e.g., 5.0"
                                className="bg-gray-700/50 border border-gray-600 rounded-md px-3 py-2 text-white placeholder-gray-500 focus:outline-none focus:ring-2 focus:ring-cyan-500 focus:border-cyan-500 transition"
                            />
                        </div>
                        {/* Duration */}
                        <div className="flex flex-col">
                            <label htmlFor="duration" className="text-sm font-medium text-gray-400 mb-1">Duration</label>
                            <input
                                id="duration"
                                type="text"
                                value={duration}
                                onChange={(e) => setDuration(e.target.value)}
                                placeholder="HH:MM:SS"
                                className="bg-gray-700/50 border border-gray-600 rounded-md px-3 py-2 text-white placeholder-gray-500 focus:outline-none focus:ring-2 focus:ring-cyan-500 focus:border-cyan-500 transition"
                            />
                        </div>
                        {/* Calories */}
                        <div className="flex flex-col">
                            <label htmlFor="calories" className="text-sm font-medium text-gray-400 mb-1">Calories</label>
                            <input
                                id="calories"
                                type="number"
                                value={calories}
                                onChange={(e) => setCalories(e.target.value)}
                                placeholder="e.g., 820"
                                className="bg-gray-700/50 border border-gray-600 rounded-md px-3 py-2 text-white placeholder-gray-500 focus:outline-none focus:ring-2 focus:ring-cyan-500 focus:border-cyan-500 transition"
                            />
                        </div>
                        {/* Submit Button */}
                        <div className="flex flex-col justify-end">
                            <button
                                type="submit"
                                className="w-full bg-gradient-to-r from-cyan-500 to-blue-600 hover:from-cyan-600 hover:to-blue-700 text-white font-bold py-2 px-4 rounded-md transition duration-300 ease-in-out transform hover:scale-105 shadow-lg"
                            >
                                Add Workout
                            </button>
                        </div>
                    </form>
                </div>

                {/* Totals Section */}
                 <div className="mb-8">
                    <h3 className="text-lg font-semibold text-gray-300 mb-3">Lifetime Totals</h3>
                    <div className="grid grid-cols-2 md:grid-cols-3 gap-4 text-center">
                        <div className="bg-white/5 backdrop-blur-sm p-4 rounded-xl border border-white/10">
                            <p className="text-sm text-gray-400">Total Distance</p>
                            <p className="text-2xl font-bold text-transparent bg-clip-text bg-gradient-to-r from-cyan-400 to-blue-400">{totalStats.distance.toFixed(2)} mi</p>
                        </div>
                        <div className="bg-white/5 backdrop-blur-sm p-4 rounded-xl border border-white/10">
                            <p className="text-sm text-gray-400">Total Time</p>
                            <p className="text-2xl font-bold text-transparent bg-clip-text bg-gradient-to-r from-cyan-400 to-blue-400">{formatDuration(totalStats.duration)}</p>
                        </div>
                        <div className="bg-white/5 backdrop-blur-sm p-4 rounded-xl border border-white/10 col-span-2 md:col-span-1">
                            <p className="text-sm text-gray-400">Total Calories</p>
                            <p className="text-2xl font-bold text-transparent bg-clip-text bg-gradient-to-r from-cyan-400 to-blue-400">{totalStats.calories.toLocaleString()}</p>
                        </div>
                    </div>
                </div>

                {/* AI Insights Section */}
                <div className="mb-8">
                    <h2 className="text-xl font-semibold mb-4 text-gray-300">✨ AI Insights</h2>
                    <div className="bg-white/5 backdrop-blur-lg p-6 rounded-2xl border border-white/10 shadow-lg">
                        <button
                            onClick={getWorkoutInsights}
                            disabled={isGenerating || workouts.length < 3}
                            className="w-full bg-gradient-to-r from-purple-500 to-indigo-600 hover:from-purple-600 hover:to-indigo-700 text-white font-bold py-2 px-4 rounded-md transition duration-300 ease-in-out transform hover:scale-105 shadow-lg disabled:opacity-50 disabled:cursor-not-allowed disabled:scale-100"
                        >
                            {isGenerating ? 'Analyzing...' : 'Generate Workout Summary & Suggestion'}
                        </button>
                        {apiError && <p className="text-red-400 text-sm mt-3">{apiError}</p>}
                        {isGenerating && (
                            <div className="text-center p-4 mt-4 text-gray-400">
                                <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-cyan-400 mx-auto"></div>
                                <p className="mt-2">Thinking...</p>
                            </div>
                        )}
                        {insights && !isGenerating && (
                            <div className="mt-4 p-4 bg-gray-700/30 rounded-lg border border-gray-600">
                               <p className="text-gray-200 whitespace-pre-wrap">{insights}</p>
                            </div>
                        )}
                         {workouts.length < 3 && !apiError && (
                            <p className="text-gray-400 text-sm mt-3 text-center">Log at least {3-workouts.length} more workout(s) to unlock AI insights!</p>
                         )}
                    </div>
                </div>


                {/* Workout History */}
                <div>
                    <h2 className="text-xl font-semibold mb-4 text-gray-300">Workout History</h2>
                     {!isAuthReady ? (
                        <div className="text-center text-gray-400">Initializing...</div>
                     ) : workouts.length === 0 ? (
                        <div className="bg-gray-800/50 p-6 rounded-xl text-center text-gray-400 border border-white/10">
                            <p>No workouts logged yet. Add your first one above!</p>
                        </div>
                    ) : (
                        <div className="space-y-4">
                            {workouts.map((workout) => (
                                <div key={workout.id} className="bg-white/5 backdrop-blur-lg p-4 rounded-xl border border-white/10 shadow-md flex flex-col sm:flex-row sm:items-center sm:justify-between transition duration-300 hover:scale-[1.02] hover:border-white/20">
                                    <div className="flex-grow mb-4 sm:mb-0">
                                        <p className="font-bold text-lg text-gray-200">{formatDate(workout.createdAt)}</p>
                                        <div className="grid grid-cols-2 sm:grid-cols-4 gap-x-4 gap-y-2 mt-2 text-sm">
                                            <div className="flex items-center" title="Distance">
                                                <RunIcon />
                                                <span className="ml-2 text-gray-300">{workout.distance.toFixed(2)} mi</span>
                                            </div>
                                            <div className="flex items-center" title="Duration">
                                                <ClockIcon />
                                                <span className="ml-2 text-gray-300">{formatDuration(workout.duration)}</span>
                                            </div>
                                            <div className="flex items-center" title="Calories">
                                                <FlameIcon />
                                                <span className="ml-2 text-gray-300">{workout.calories ? `${workout.calories} kcal` : '--'}</span>
                                            </div>
                                            <div className="flex items-center" title="Pace per mile">
                                                <GaugeIcon />
                                                <span className="ml-2 text-gray-300">{calculatePace(workout.distance, formatDuration(workout.duration))} /mi</span>
                                            </div>
                                        </div>
                                    </div>
                                    <div className="flex-shrink-0">
                                        <button 
                                            onClick={() => handleDelete(workout.id)}
                                            className="p-2 rounded-full text-gray-400 hover:bg-red-500/20 hover:text-red-400 transition-colors"
                                            aria-label="Delete workout"
                                        >
                                            <TrashIcon className="h-5 w-5"/>
                                        </button>
                                    </div>
                                </div>
                            ))}
                        </div>
                    )}
                </div>
            </div>
        </div>
    );
}



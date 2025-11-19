import React, { useState, useEffect, useCallback, useMemo } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInWithCustomToken, signInAnonymously, onAuthStateChanged, setPersistence, browserLocalPersistence } from 'firebase/auth';
import { getFirestore, doc, setDoc, collection, query, onSnapshot, updateDoc, deleteDoc } from 'firebase/firestore';
import { 
  Home, User, LogOut, CheckSquare, Settings, AlertTriangle, MessageSquare, Clock, Edit, FileText, X, Trash2, Plus, 
  Brain, Heart, Cloud, Droplet 
} from 'lucide-react';

// --- Global Firebase Variable Setup (MANDATORY) ---
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

// Initial patient data structure for a new patient
const initialPatientData = {
  roomNumber: '',
  name: 'New Patient',
  age: 0,
  codeStatus: 'Full Code',
  isolationType: 'None',
  quickOneLiner: 'Needs documentation.',
  admissionDate: new Date().toISOString().split('T')[0],
  primaryDiagnosis: '',
  allergies: [],
  pastMedicalHistory: [],
  hospitalCourse: [],
  systemsReview: {}, // Populated by config
  todoList: [],
  notes: '',
  lastUpdated: new Date().toISOString(),
};

// Configuration for the Systems Review checklist
const systemsChecklistConfig = [
  { id: 'neuro', name: 'Neurological', icon: 'Brain', defaultStatus: 'Alert & Oriented x 3', items: [
    { key: 'aox3', label: 'Alert & Oriented x 3' },
    { key: 'pupils', label: 'Pupils Equal/Reactive' },
    { key: 'grips', label: 'Strong/Equal Grips' },
  ]},
  { id: 'cardio', name: 'Cardiovascular', icon: 'Heart', defaultStatus: 'Regular Rate & Rhythm', items: [
    { key: 'tele', label: 'Telemetry/Monitor' },
    { key: 'pulses', label: '2+ Pedal Pulses' },
    { key: 'edema', label: 'No Edema' },
  ]},
  { id: 'resp', name: 'Respiratory', icon: 'Cloud', defaultStatus: 'Clear bilateral breath sounds', items: [
    { key: 'o2', label: 'On Supplemental O2' },
    { key: 'cough', label: 'Productive Cough' },
    { key: 'bs', label: 'Diminished Breath Sounds' },
  ]},
  { id: 'gi', name: 'GI/GU', icon: 'Droplet', defaultStatus: 'Soft/Nontender abdomen, voiding w/o issue', items: [
    { key: 'npo', label: 'NPO Status' },
    { key: 'bm', label: 'BM Today' },
    { key: 'foley', label: 'Foley/Catheter' },
  ]},
];


// --- FIREBASE INITIALIZATION & AUTH HOOK ---
function useFirebase() {
  const [db, setDb] = useState(null);
  const [auth, setAuth] = useState(null);
  const [userId, setUserId] = useState(null);
  const [isAuthReady, setIsAuthReady] = useState(false);

  useEffect(() => {
    try {
      if (Object.keys(firebaseConfig).length === 0) {
        console.error("Firebase config is empty. Check __firebase_config.");
        return;
      }

      const app = initializeApp(firebaseConfig);
      const firestoreDb = getFirestore(app);
      const firebaseAuth = getAuth(app);
      setDb(firestoreDb);
      setAuth(firebaseAuth);

      const unsubscribe = onAuthStateChanged(firebaseAuth, async (user) => {
        if (user) {
          setUserId(user.uid);
        } else {
          if (initialAuthToken) {
            await signInWithCustomToken(firebaseAuth, initialAuthToken);
          } else {
            await setPersistence(firebaseAuth, browserLocalPersistence);
            await signInAnonymously(firebaseAuth);
          }
        }
        setIsAuthReady(true);
      });

      return () => unsubscribe();
    } catch (e) {
      console.error("Firebase initialization failed:", e);
      setIsAuthReady(true);
    }
  }, []);

  return { db, auth, userId, isAuthReady };
}

// --- FIREBASE DATA HOOK ---
function usePatientData(db, userId, isAuthReady) {
  const [patients, setPatients] = useState([]);
  const [isLoading, setIsLoading] = useState(true);

  // Firestore path for private user data
  const getPatientCollectionPath = useCallback(() => {
    if (userId) {
      return `artifacts/${appId}/users/${userId}/patients`;
    }
    return null;
  }, [userId]);

  useEffect(() => {
    if (!db || !isAuthReady || !userId) {
      if (isAuthReady) setIsLoading(false);
      return;
    }

    const path = getPatientCollectionPath();
    if (!path) return;

    const patientsQuery = query(collection(db, path));

    const unsubscribe = onSnapshot(patientsQuery, (snapshot) => {
      const patientList = snapshot.docs.map(doc => ({
        id: doc.id,
        ...doc.data()
      }));
      setPatients(patientList.sort((a, b) => (a.roomNumber || '').localeCompare(b.roomNumber || '')));
      setIsLoading(false);
    }, (error) => {
      console.error("Error fetching patients:", error);
      setIsLoading(false);
    });

    return () => unsubscribe();
  }, [db, userId, isAuthReady, getPatientCollectionPath]);

  const updatePatientData = useCallback(async (patientId, data) => {
    if (!db || !userId) {
      console.error("DB or User not ready.");
      return;
    }
    try {
      const path = getPatientCollectionPath();
      if (!path) return;

      const patientRef = doc(db, path, patientId);
      await updateDoc(patientRef, { ...data, lastUpdated: new Date().toISOString() });
    } catch (error) {
      console.error("Error updating patient:", error);
    }
  }, [db, userId, getPatientCollectionPath]);

  // REFACTORED: Now accepts roomNumber and patientName directly, no prompts
  const addPatient = useCallback(async (roomNumber, patientName) => {
    if (!db || !userId) {
      console.error("DB or User not ready.");
      return;
    }
    
    // Check if required fields are provided
    if (!roomNumber || !patientName) {
        console.warn("Room number and name are required for new patient.");
        return;
    }

    try {
      const path = getPatientCollectionPath();
      if (!path) return;

      const patientRef = doc(collection(db, path));
      const initialSystems = systemsChecklistConfig.reduce((acc, sys) => {
        acc[sys.id] = {
          checklist: sys.items.reduce((itemAcc, item) => ({ ...itemAcc, [item.key]: true }), {}),
          notes: '',
        };
        return acc;
      }, {});

      const newPatient = {
        ...initialPatientData,
        roomNumber: roomNumber,
        name: patientName,
        systemsReview: initialSystems,
      };

      await setDoc(patientRef, newPatient);
    } catch (error) {
      console.error("Error adding new patient:", error);
    }
  }, [db, userId, getPatientCollectionPath]);

  // REFACTORED: Now accepts patientId directly, confirmation is handled by modal in App
  const deletePatient = useCallback(async (patientId) => {
    if (!db || !userId) {
      console.error("DB or User not ready.");
      return;
    }
    try {
      const path = getPatientCollectionPath();
      if (!path) return;

      const patientRef = doc(db, path, patientId);
      await deleteDoc(patientRef);
    } catch (error) {
      console.error("Error deleting patient:", error);
    }
  }, [db, userId, getPatientCollectionPath]);

  return { patients, isLoading, updatePatientData, addPatient, deletePatient };
}

// --- GENERIC MODAL COMPONENT (Replaces all browser prompts/confirms) ---
const Modal = ({ isOpen, onClose, title, children }) => {
    if (!isOpen) return null;

    return (
        <div className="fixed inset-0 bg-black bg-opacity-50 z-50 flex items-center justify-center p-4">
            <div className="bg-white rounded-xl shadow-2xl w-full max-w-md max-h-[90vh] overflow-y-auto">
                <div className="flex justify-between items-center p-4 border-b">
                    <h3 className="text-xl font-bold text-gray-800">{title}</h3>
                    <button onClick={onClose} className="text-gray-400 hover:text-gray-600">
                        <X className="w-6 h-6" />
                    </button>
                </div>
                <div className="p-6">
                    {children}
                </div>
            </div>
        </div>
    );
};

// --- MODAL FORMS ---

// 1. Add Patient Form
const AddPatientForm = ({ onSubmit, onClose }) => {
    const [roomNumber, setRoomNumber] = useState('');
    const [patientName, setPatientName] = useState('');
    const [age, setAge] = useState('');

    const handleSubmit = (e) => {
        e.preventDefault();
        if (roomNumber.trim() && patientName.trim()) {
            onSubmit({ roomNumber: roomNumber.trim(), patientName: patientName.trim(), age: parseInt(age) || 0 });
        }
    };

    return (
        <form onSubmit={handleSubmit} className="space-y-4">
            <div>
                <label className="block text-sm font-medium text-gray-700">Room Number (e.g., 305A)</label>
                <input
                    type="text"
                    required
                    value={roomNumber}
                    onChange={(e) => setRoomNumber(e.target.value)}
                    className="mt-1 w-full p-3 border border-gray-300 rounded-lg focus:ring-blue-500 focus:border-blue-500"
                    placeholder="Required"
                />
            </div>
            <div>
                <label className="block text-sm font-medium text-gray-700">Patient Name</label>
                <input
                    type="text"
                    required
                    value={patientName}
                    onChange={(e) => setPatientName(e.target.value)}
                    className="mt-1 w-full p-3 border border-gray-300 rounded-lg focus:ring-blue-500 focus:border-blue-500"
                    placeholder="Required"
                />
            </div>
            <div>
                <label className="block text-sm font-medium text-gray-700">Age</label>
                <input
                    type="number"
                    value={age}
                    onChange={(e) => setAge(e.target.value)}
                    className="mt-1 w-full p-3 border border-gray-300 rounded-lg focus:ring-blue-500 focus:border-blue-500"
                    placeholder="Optional"
                />
            </div>
            <div className="flex justify-end space-x-3 pt-4">
                <button type="button" onClick={onClose} className="px-4 py-2 text-sm font-medium text-gray-700 bg-gray-100 rounded-lg hover:bg-gray-200 transition">
                    Cancel
                </button>
                <button type="submit" className="px-4 py-2 text-sm font-medium text-white bg-blue-600 rounded-lg hover:bg-blue-700 transition">
                    <Plus className="w-4 h-4 inline mr-1" />
                    Add Patient
                </button>
            </div>
        </form>
    );
};

// 2. Generic Input Form (For To-Do/Hospital Course)
const GenericInputForm = ({ onSubmit, onClose, label, placeholder }) => {
    const [input, setInput] = useState('');

    const handleSubmit = (e) => {
        e.preventDefault();
        if (input.trim()) {
            onSubmit(input.trim());
        }
    };

    return (
        <form onSubmit={handleSubmit} className="space-y-4">
            <div>
                <label className="block text-sm font-medium text-gray-700">{label}</label>
                <textarea
                    rows="3"
                    required
                    value={input}
                    onChange={(e) => setInput(e.target.value)}
                    className="mt-1 w-full p-3 border border-gray-300 rounded-lg focus:ring-blue-500 focus:border-blue-500 resize-none"
                    placeholder={placeholder}
                />
            </div>
            <div className="flex justify-end space-x-3 pt-4">
                <button type="button" onClick={onClose} className="px-4 py-2 text-sm font-medium text-gray-700 bg-gray-100 rounded-lg hover:bg-gray-200 transition">
                    Cancel
                </button>
                <button type="submit" className="px-4 py-2 text-sm font-medium text-white bg-blue-600 rounded-lg hover:bg-blue-700 transition">
                    <Plus className="w-4 h-4 inline mr-1" />
                    Add Item
                </button>
            </div>
        </form>
    );
};

// 3. Delete Confirmation
const DeleteConfirmation = ({ onConfirm, onClose, patientName }) => (
    <div className="space-y-6">
        <p className="text-gray-700">
            Are you sure you want to **discharge/delete** patient **{patientName}**? 
            This action is permanent and will remove them from your census.
        </p>
        <div className="flex justify-end space-x-3 pt-4">
            <button onClick={onClose} className="px-4 py-2 text-sm font-medium text-gray-700 bg-gray-100 rounded-lg hover:bg-gray-200 transition">
                Cancel
            </button>
            <button onClick={onConfirm} className="px-4 py-2 text-sm font-medium text-white bg-red-600 rounded-lg hover:bg-red-700 transition">
                <Trash2 className="w-4 h-4 inline mr-1" />
                Confirm Discharge
            </button>
        </div>
    </div>
);


// 4. Report Output (Replaces window.prompt for SBAR)
const SBARReportOutput = ({ patient, onClose }) => {
    
    const reportText = useMemo(() => 
        `S: ${patient.quickOneLiner}
B: Patient is a ${patient.age} y.o. admitted for ${patient.primaryDiagnosis}. Allergies: ${patient.allergies.join(', ') || 'None'}.
A: Key Systems Review: 
${systemsChecklistConfig.map(sys => 
    `  - ${sys.name}: ${patient.systemsReview[sys.id]?.notes || sys.defaultStatus}`
).join('\n')}
R: To-Do: ${patient.todoList.filter(t => !t.completed).map(t => t.task).join('; ') || 'None outstanding.'}`
    , [patient]);

    const handleCopy = () => {
        const textarea = document.createElement('textarea');
        textarea.value = reportText;
        document.body.appendChild(textarea);
        textarea.select();
        try {
            document.execCommand('copy');
            alert('SBAR Report copied to clipboard!'); // Using simple alert for feedback on copy action
        } catch (err) {
            console.error('Could not copy text: ', err);
            alert('Failed to copy. Please manually copy the text below.');
        }
        document.body.removeChild(textarea);
    };

    return (
        <div className="space-y-4">
            <p className="text-sm text-gray-600">Review and copy the complete SBAR report for transfer of care:</p>
            <textarea
                readOnly
                rows="10"
                value={reportText}
                className="w-full p-3 border border-gray-300 rounded-lg bg-gray-50 text-sm font-mono resize-none"
            />
            <div className="flex justify-end space-x-3 pt-4">
                <button onClick={onClose} className="px-4 py-2 text-sm font-medium text-gray-700 bg-gray-100 rounded-lg hover:bg-gray-200 transition">
                    Done
                </button>
                <button onClick={handleCopy} className="px-4 py-2 text-sm font-medium text-white bg-green-600 rounded-lg hover:bg-green-700 transition">
                    <MessageSquare className="w-4 h-4 inline mr-1" />
                    Copy Report
                </button>
            </div>
        </div>
    );
};


// --- UI COMPONENTS (Minimal changes, mostly removing window.prompt) ---

// 1. Loading/Auth State
const AuthLoading = ({ isAuthReady }) => (
  <div className="flex flex-col items-center justify-center min-h-screen bg-gray-100 p-4">
    <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-blue-500"></div>
    <p className="mt-4 text-gray-700 font-semibold">
      {isAuthReady ? 'Authenticating User...' : 'Initializing App...'}
    </p>
    <p className="text-sm text-gray-500 mt-1">
      Securing connection to the unit database.
    </p>
  </div>
);

// 2. Patient Card (For Census View)
const PatientCard = ({ patient, onSelect, onDelete }) => {
    const statusColor = patient.codeStatus === 'DNR' ? 'bg-red-100 text-red-700 border-red-300' : 'bg-green-100 text-green-700 border-green-300';
    const isolationColor = patient.isolationType !== 'None' ? 'text-red-600 bg-red-100 p-1 rounded-full' : 'text-gray-500';

    return (
        <div className="bg-white rounded-xl shadow-lg hover:shadow-xl transition-all duration-300 border border-gray-100 mb-4 overflow-hidden">
            <div className={`p-4 border-l-8 ${statusColor.includes('red') ? 'border-red-500' : 'border-green-500'}`}>
                <div className="flex justify-between items-start mb-2">
                    <h2 className="text-2xl font-extrabold text-gray-900">Room {patient.roomNumber || 'N/A'}</h2>
                    <span className={`text-sm font-bold ${isolationColor}`}>
                        {patient.isolationType !== 'None' && <AlertTriangle className="inline w-4 h-4 mr-1" />}
                        {patient.isolationType}
                    </span>
                </div>
                <h3 className="text-xl font-semibold text-gray-700">{patient.name} ({patient.age} y.o.)</h3>
                <p className={`text-xs font-bold mt-1 inline-block px-2 py-0.5 rounded-lg ${statusColor}`}>
                    {patient.codeStatus}
                </p>
                <p className="text-sm text-gray-500 mt-2 italic line-clamp-2">{patient.quickOneLiner || 'No quick summary documented.'}</p>
                
                <div className="mt-4 flex space-x-3">
                    <button 
                        onClick={() => onSelect(patient)}
                        className="flex-1 flex items-center justify-center bg-blue-600 text-white py-2 rounded-lg font-medium hover:bg-blue-700 transition duration-150 shadow-md"
                    >
                        <FileText className="w-4 h-4 mr-2" />
                        Start Report
                    </button>
                    <button 
                        onClick={(e) => { e.stopPropagation(); onDelete(patient.id, patient.name); }} // PASSING NAME NOW
                        className="p-2 bg-gray-200 text-gray-600 rounded-lg hover:bg-red-500 hover:text-white transition duration-150"
                        title="Discharge Patient"
                    >
                        <LogOut className="w-5 h-5" />
                    </button>
                </div>
            </div>
        </div>
    );
};

// 3. Systems Review Sub-Component (No change needed here)
const SystemReviewInput = ({ system, patientData, updatePatientData }) => {
    
    const IconComponent = ({ name }) => {
        const icons = { Brain: Brain, Heart: Heart, Cloud: Cloud, Droplet: Droplet };
        const SelectedIcon = icons[name] || Settings; 
        return <SelectedIcon className="w-5 h-5 mr-3 text-blue-500" />;
    };

    const currentSystemData = patientData.systemsReview[system.id] || { checklist: {}, notes: '' };
    
    const summary = useMemo(() => {
        const normalChecks = system.items.filter(item => currentSystemData.checklist[item.key] !== false).length;
        const totalChecks = system.items.length;
        const isNormal = normalChecks === totalChecks;

        if (isNormal && !currentSystemData.notes) return system.defaultStatus;
        
        let status = '';
        if (normalChecks < totalChecks) {
            const deviations = system.items.filter(item => currentSystemData.checklist[item.key] === false).map(item => item.label.split(' ')[0]).join(', ');
            status += `${totalChecks - normalChecks} deviations (${deviations})`;
        } else {
            status += 'Checks WNL.';
        }
        
        return status + (currentSystemData.notes ? ` Notes: ${currentSystemData.notes.substring(0, 30)}...` : '');
    }, [currentSystemData, system]);


    const handleToggle = (key) => {
        const newChecklist = { ...currentSystemData.checklist, [key]: !currentSystemData.checklist[key] };
        const newSystemsReview = { ...patientData.systemsReview, [system.id]: { ...currentSystemData, checklist: newChecklist } };
        updatePatientData(patientData.id, { systemsReview: newSystemsReview });
    };

    const handleNotesChange = (e) => {
        const newSystemsReview = { ...patientData.systemsReview, [system.id]: { ...currentSystemData, notes: e.target.value } };
        updatePatientData(patientData.id, { systemsReview: newSystemsReview });
    };

    return (
        <details className="mb-4 rounded-xl shadow-md overflow-hidden bg-white border border-gray-200 transition-all duration-300 open:shadow-xl">
            <summary className="flex items-center p-4 cursor-pointer bg-blue-50 hover:bg-blue-100 transition duration-150">
                <IconComponent name={system.icon} />
                <div className="flex-1">
                    <h4 className="font-bold text-gray-800">{system.name}</h4>
                    <p className="text-sm text-gray-600 mt-0.5 italic line-clamp-1">{summary}</p>
                </div>
                <span className="text-xs text-blue-600 ml-4 font-semibold">Click to Edit</span>
            </summary>
            
            <div className="p-4 border-t border-gray-100">
                {/* Checklist */}
                <div className="space-y-2 mb-4">
                    {system.items.map(item => (
                        <div key={item.key} className="flex justify-between items-center py-1.5 border-b border-gray-50 last:border-b-0">
                            <label className="text-gray-700">{item.label}</label>
                            <input 
                                type="checkbox" 
                                checked={currentSystemData.checklist[item.key] || false}
                                onChange={() => handleToggle(item.key)}
                                className="h-5 w-5 text-blue-600 border-gray-300 rounded focus:ring-blue-500"
                            />
                        </div>
                    ))}
                </div>

                {/* Notes Input */}
                <textarea
                    rows="3"
                    className="w-full p-3 border border-gray-300 rounded-lg focus:ring-blue-500 focus:border-blue-500 text-sm resize-none"
                    placeholder={`Detailed notes for ${system.name} (e.g., patient refused, deviation details)`}
                    value={currentSystemData.notes}
                    onChange={handleNotesChange}
                />
            </div>
        </details>
    );
};

// 4. Full Patient Profile / SBAR View
const PatientProfile = ({ patient, updatePatientData, onBack, onOpenModal }) => {
    // Helper to update top-level patient fields
    const handleFieldChange = (field, value) => {
        updatePatientData(patient.id, { [field]: value });
    };

    // Helper for List-based entries (To-Do, Hospital Course)
    const handleListAdd = (field, newItem) => {
        const list = patient[field] || [];
        const newList = [...list, newItem];
        updatePatientData(patient.id, { [field]: newList });
    };

    const handleListDelete = (field, index) => {
        const list = patient[field] || [];
        const newList = list.filter((_, i) => i !== index);
        updatePatientData(patient.id, { [field]: newList });
    };

    // REFACTORED: Now uses modal for input
    const handleAddMajorEventClick = () => {
        onOpenModal('input', 'Add Major Hospital Event', ({ input }) => {
            handleListAdd('hospitalCourse', { date: new Date().toLocaleDateString(), event: input });
        }, 'Enter major event (e.g., "Procedure: MRI done, negative results")');
    };

    // REFACTORED: Now uses modal for input
    const handleAddTodoItemClick = () => {
        onOpenModal('input', 'Add To-Do Item', ({ input }) => {
            handleListAdd('todoList', { task: input, completed: false });
        }, "Enter new task for the shift (e.g., 'Check blood sugar at 2200'):");
    };

    // REFACTORED: Now uses modal for SBAR output
    const handlePrintReportClick = () => {
        onOpenModal('report', 'SBAR Report Generation', patient);
    };


    return (
        <div className="p-4 md:p-6 bg-gray-50 min-h-screen">
            <button 
                onClick={onBack}
                className="flex items-center text-blue-600 hover:text-blue-800 mb-6 font-medium transition duration-150"
            >
                &larr; Back to Census
            </button>

            <div className="bg-white rounded-xl shadow-2xl p-6 mb-8 border-t-4 border-blue-600">
                <div className="flex justify-between items-start">
                    <div>
                        <h1 className="text-3xl font-extrabold text-gray-900">{patient.name}</h1>
                        <p className="text-lg text-gray-600">Room {patient.roomNumber} | Age {patient.age}</p>
                    </div>
                    <div className="text-right">
                        <p className={`font-bold text-lg ${patient.codeStatus === 'DNR' ? 'text-red-600' : 'text-green-600'}`}>
                            {patient.codeStatus}
                        </p>
                        <p className="text-sm text-gray-500">Last Updated: {new Date(patient.lastUpdated).toLocaleTimeString()}</p>
                    </div>
                </div>
            </div>

            {/* Quick One-Liner (S) */}
            <h2 className="text-2xl font-bold text-gray-800 mb-4 flex items-center"><FileText className="w-5 h-5 mr-2"/>Situation (S)</h2>
            <div className="mb-6">
                <textarea
                    rows="2"
                    value={patient.quickOneLiner}
                    onChange={(e) => handleFieldChange('quickOneLiner', e.target.value)}
                    className="w-full p-3 border border-gray-300 rounded-lg focus:ring-blue-500 focus:border-blue-500 text-base"
                    placeholder="Brief summary of status and most critical issue."
                />
            </div>

            {/* Background & History (B) */}
            <h2 className="text-2xl font-bold text-gray-800 mb-4 flex items-center"><User className="w-5 h-5 mr-2"/>Background (B)</h2>
            <div className="grid md:grid-cols-2 gap-4 mb-6">
                <input
                    type="text"
                    placeholder="Primary Diagnosis"
                    value={patient.primaryDiagnosis}
                    onChange={(e) => handleFieldChange('primaryDiagnosis', e.target.value)}
                    className="p-3 border border-gray-300 rounded-lg"
                />
                <input
                    type="text"
                    placeholder="Allergies (comma separated)"
                    value={patient.allergies.join(', ')}
                    onChange={(e) => handleFieldChange('allergies', e.target.value.split(',').map(s => s.trim()))}
                    className="p-3 border border-gray-300 rounded-lg"
                />
            </div>
            
            {/* Systems Review (A) */}
            <h2 className="text-2xl font-bold text-gray-800 mb-4 flex items-center"><CheckSquare className="w-5 h-5 mr-2"/>Assessment (A)</h2>
            <div className="mb-6 space-y-4">
                {systemsChecklistConfig.map(system => (
                    <SystemReviewInput 
                        key={system.id} 
                        system={system} 
                        patientData={patient} 
                        updatePatientData={updatePatientData} 
                    />
                ))}
            </div>

            {/* Hospital Course / Timeline */}
            <h2 className="text-2xl font-bold text-gray-800 mb-4 flex items-center"><Clock className="w-5 h-5 mr-2"/>Hospital Course</h2>
            <div className="bg-white p-4 rounded-xl shadow-md mb-6">
                <ul className="space-y-3">
                    {(patient.hospitalCourse || []).map((item, index) => (
                        <li key={index} className="flex justify-between items-start text-sm text-gray-700 border-b pb-2 last:border-b-0">
                            <span className="font-semibold mr-3">{item.date}:</span>
                            <span className="flex-1">{item.event}</span>
                            <button onClick={() => handleListDelete('hospitalCourse', index)} className="text-red-500 hover:text-red-700 ml-3">
                                <Trash2 className="w-4 h-4"/>
                            </button>
                        </li>
                    ))}
                </ul>
                <button 
                    onClick={handleAddMajorEventClick}
                    className="mt-4 w-full text-center py-2 text-sm bg-gray-100 text-gray-600 rounded-lg hover:bg-gray-200 transition"
                >
                    + Add Major Event
                </button>
            </div>

            {/* To-Do / Recommendations (R) */}
            <h2 className="text-2xl font-bold text-gray-800 mb-4 flex items-center"><Edit className="w-5 h-5 mr-2"/>Recommendation (R)</h2>
            <div className="bg-white p-4 rounded-xl shadow-md mb-8">
                <ul className="space-y-2">
                    {(patient.todoList || []).map((item, index) => (
                        <li key={index} className="flex justify-between items-center text-base text-gray-800">
                            <label className="flex items-center space-x-2">
                                <input 
                                    type="checkbox" 
                                    checked={item.completed}
                                    onChange={() => {
                                        const newList = patient.todoList.map((t, i) => i === index ? { ...t, completed: !t.completed } : t);
                                        handleFieldChange('todoList', newList);
                                    }}
                                    className="h-4 w-4 text-blue-600 border-gray-300 rounded"
                                />
                                <span className={item.completed ? 'line-through text-gray-500' : 'font-medium'}>{item.task}</span>
                            </label>
                            <button onClick={() => handleListDelete('todoList', index)} className="text-red-500 hover:text-red-700 ml-3">
                                &times;
                            </button>
                        </li>
                    ))}
                </ul>
                <button 
                    onClick={handleAddTodoItemClick}
                    className="mt-4 w-full text-center py-2 text-sm bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition"
                >
                    + Add To-Do Item
                </button>
            </div>
            
            {/* Print / Share Button */}
            <div className="fixed bottom-0 left-0 right-0 p-4 bg-white border-t shadow-2xl">
                <button 
                    onClick={handlePrintReportClick}
                    className="w-full flex items-center justify-center bg-green-600 text-white py-3 rounded-xl font-bold text-lg hover:bg-green-700 transition duration-300"
                >
                    <MessageSquare className="w-5 h-5 mr-2" />
                    Give Report / Copy SBAR
                </button>
            </div>
            
            <div className="h-20"></div> {/* Spacer for fixed bottom bar */}
        </div>
    );
};


// 5. Main App Component
const App = () => {
  const { db, userId, isAuthReady } = useFirebase();
  const { patients, isLoading, updatePatientData, addPatient, deletePatient } = usePatientData(db, userId, isAuthReady);
  const [selectedPatient, setSelectedPatient] = useState(null);

  // --- MODAL STATE MANAGEMENT ---
  const [isModalOpen, setIsModalOpen] = useState(false);
  const [modalContent, setModalContent] = useState({});

  const closeModal = () => {
    setIsModalOpen(false);
    setModalContent({});
  };
  
  // Generic handler to open different modal types
  const openModal = useCallback((type, title, data, placeholder = null) => {
    setModalContent({ type, title, data, placeholder });
    setIsModalOpen(true);
  }, []);

  // Handlers for Census Page buttons
  const handleOpenAddPatientModal = () => {
    openModal('addPatient', 'Add New Patient', {}, null);
  };
  
  const handleDeletePatientClick = (patientId, patientName) => {
    openModal('confirmDelete', 'Confirm Discharge', { patientId, patientName });
  };
  
  const handleConfirmDelete = (patientId) => {
    deletePatient(patientId);
    closeModal();
  };

  // Render logic for the different modal types
  const renderModalContent = () => {
    const { type, title, data, placeholder } = modalContent;
    
    return (
        <Modal isOpen={isModalOpen} onClose={closeModal} title={title}>
            {type === 'addPatient' && (
                <AddPatientForm 
                    onClose={closeModal}
                    onSubmit={({ roomNumber, patientName, age }) => {
                        addPatient(roomNumber, patientName, age);
                        closeModal();
                    }}
                />
            )}
            {type === 'confirmDelete' && (
                <DeleteConfirmation 
                    onClose={closeModal}
                    patientName={data.patientName}
                    onConfirm={() => handleConfirmDelete(data.patientId)}
                />
            )}
            {type === 'input' && (
                <GenericInputForm 
                    onClose={closeModal}
                    label={title}
                    placeholder={placeholder}
                    onSubmit={(input) => {
                        // onSubmit on the modal content is the actual list adding function
                        // data holds the required context for the list action
                        if (data && data.input) data.input({ input });
                        closeModal();
                    }}
                />
            )}
            {type === 'report' && (
                <SBARReportOutput 
                    onClose={closeModal}
                    patient={data}
                />
            )}
        </Modal>
    );
  };

  if (!isAuthReady || isLoading) {
    return <AuthLoading isAuthReady={isAuthReady} />;
  }
  
  // App Router (simple state-based routing)
  if (selectedPatient) {
    return (
      <>
        <PatientProfile 
          patient={selectedPatient} 
          updatePatientData={updatePatientData}
          onBack={() => setSelectedPatient(null)}
          onOpenModal={(type, title, data, placeholder) => {
            // Special handling for nested input modal actions
            if (type === 'input') {
                openModal('input', title, { input: data }, placeholder);
            } else if (type === 'report') {
                openModal('report', title, data);
            }
          }}
        />
        {renderModalContent()}
      </>
    );
  }

  // Home Page: Patient Census
  return (
    <div className="p-4 md:p-6 bg-gray-100 min-h-screen">
      <header className="flex justify-between items-center mb-6 border-b pb-3 sticky top-0 z-10 bg-gray-100">
        <div className="flex items-center">
            <Home className="w-6 h-6 text-blue-600 mr-2" />
            <h1 className="text-3xl font-extrabold text-gray-800">Today's Census</h1>
        </div>
        <div className="flex items-center">
            <span className="text-xs md:text-sm text-gray-500 mr-2 font-mono truncate max-w-[100px] md:max-w-none">
                {userId}
            </span>
            <User className="w-5 h-5 text-gray-500" />
        </div>
      </header>

      {patients.length === 0 ? (
        <div className="text-center p-10 bg-white rounded-xl shadow-lg border border-dashed border-gray-300">
          <FileText className="w-10 h-10 text-gray-400 mx-auto mb-4" />
          <p className="text-lg font-semibold text-gray-700">No patients assigned yet.</p>
          <p className="text-sm text-gray-500 mt-2">Tap below to add your first patient and begin charting.</p>
        </div>
      ) : (
        <div className="space-y-4">
          {patients.map(p => (
            <PatientCard 
              key={p.id} 
              patient={p} 
              onSelect={setSelectedPatient} 
              onDelete={handleDeletePatientClick}
            />
          ))}
        </div>
      )}

      {/* Fixed Add Button */}
      <button 
        onClick={handleOpenAddPatientModal}
        className="fixed bottom-6 right-6 flex items-center p-4 bg-blue-600 text-white rounded-full shadow-2xl hover:bg-blue-700 transition duration-200 z-20 font-bold text-lg"
        title="Add New Patient"
      >
        <CheckSquare className="w-6 h-6 mr-2" />
        Add Patient
      </button>

      <div className="h-20"></div> {/* Spacer for bottom of content */}
      
      {renderModalContent()}
    </div>
  );
};

export default App;

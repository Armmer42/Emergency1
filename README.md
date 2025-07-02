import React, { useState, useEffect } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, collection, addDoc, onSnapshot, query, orderBy, serverTimestamp } from 'firebase/firestore';

// Firebase configuration and initialization (provided by the environment)
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';

// Initialize Firebase app outside of the component to avoid re-initialization
let app;
let db;
let auth;

try {
    app = initializeApp(firebaseConfig);
    db = getFirestore(app);
    auth = getAuth(app);
} catch (error) {
    console.error("Error initializing Firebase:", error);
}

// Main App component
const App = () => {
    const [userId, setUserId] = useState(null);
    const [isAuthReady, setIsAuthReady] = useState(false);
    const [accidents, setAccidents] = useState([]);

    // State variables for form fields
    const [name, setName] = useState('');
    const [faculty, setFaculty] = useState('คณะเกษตร กำแพงแสน'); // Default to the first option
    const [year, setYear] = useState('ปีที่ 1'); // Default to the first year option
    const [accidentType, setAccidentType] = useState('อุบัติเหตุจราจร'); // Default value
    const [location, setLocation] = useState('');
    const [description, setDescription] = useState('');
    const [severity, setSeverity] = useState('เล็กน้อย');
    const [injuries, setInjuries] = useState(0);
    const [remarks, setRemarks] = useState(''); // Field for remarks

    const [showModal, setShowModal] = useState(false);
    const [modalMessage, setModalMessage] = useState('');
    const [selectedFile, setSelectedFile] = useState(null); // State for uploaded file
    const [isImporting, setIsImporting] = useState(false); // Loading state for import

    // State variables for date range filtering
    const [startDate, setStartDate] = useState('');
    const [endDate, setEndDate] = useState('');

    // State variables for AI analysis
    const [aiAnalysisInput, setAiAnalysisInput] = useState('');
    const [aiAnalysisOutput, setAiAnalysisOutput] = useState('ผลการวิเคราะห์จาก AI จะแสดงที่นี่...');
    const [isAiLoading, setIsAiLoading] = useState(false);

    // List of faculties for the dropdown
    const faculties = [
        "คณะเกษตร กำแพงแสน",
        "คณะวิศวกรรมศาสตร์ กำแพงแสน",
        "คณะวิทยาศาสตร์การกีฬาและสุขภาพ",
        "คณะศึกษาศาสตร์และพัฒนศาสตร์",
        "คณะศิลปศาสตร์และวิทยาศาสตร์",
        "คณะสัตวแพทยศาสตร์",
        "คณะประมง",
        "คณะสิ่งแวดล้อม",
        "คณะอุตสาหกรรมบริการ",
        "บุคลากร",
        "อื่นๆ"
    ];

    // List of year levels/positions for the dropdown
    const yearLevels = [
        "ปีที่ 1",
        "ปีที่ 2",
        "ปีที่ 3",
        "ปีที่ 4",
        "ปีที่ 5",
        "ปีที่ 6",
        "ระดับบัณฑิตศึกษา",
        "บุคลากร/พนักงาน",
        "อื่นๆ"
    ];

    // Authentication and Firestore setup
    useEffect(() => {
        if (!auth) {
            console.error("Firebase Auth is not initialized.");
            return;
        }

        const unsubscribe = onAuthStateChanged(auth, async (user) => {
            if (user) {
                setUserId(user.uid);
            } else {
                // Sign in anonymously if no user is authenticated
                try {
                    const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
                    if (initialAuthToken) {
                        await signInWithCustomToken(auth, initialAuthToken);
                    } else {
                        await signInAnonymously(auth);
                    }
                    setUserId(auth.currentUser?.uid || crypto.randomUUID()); // Fallback for userId
                } catch (error) {
                    console.error("Error signing in:", error);
                    setUserId(crypto.randomUUID()); // Use a random ID if sign-in fails
                }
            }
            setIsAuthReady(true);
        });

        return () => unsubscribe();
    }, []);

    // Fetch accidents from Firestore
    useEffect(() => {
        if (!db || !userId || !isAuthReady) {
            return;
        }

        const accidentsCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/accidents`);
        
        const q = query(accidentsCollectionRef, orderBy('timestamp', 'desc'));

        const unsubscribe = onSnapshot(q, (snapshot) => {
            const accidentsData = snapshot.docs.map(doc => ({
                id: doc.id,
                ...doc.data()
            }));
            setAccidents(accidentsData);
        }, (error) => {
            console.error("Error fetching accidents:", error);
            showUserMessage("เกิดข้อผิดพลาดในการดึงข้อมูลอุบัติเหตุ");
        });

        return () => unsubscribe();
    }, [db, userId, isAuthReady]);

    // Function to show a modal message to the user
    const showUserMessage = (message) => {
        setModalMessage(message);
        setShowModal(true);
    };

    // Handle form submission
    const handleSubmit = async (e) => {
        e.preventDefault();
        if (!db || !userId) {
            showUserMessage("ระบบยังไม่พร้อม กรุณารอสักครู่");
            return;
        }

        // Basic validation for required fields
        if (!name || !faculty || !year || !location || !description) {
            showUserMessage("กรุณากรอกข้อมูลให้ครบถ้วน (ชื่อ, คณะ, ชั้นปี/ตำแหน่ง, สถานที่, รายละเอียด)");
            return;
        }

        try {
            const accidentsCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/accidents`);
            await addDoc(accidentsCollectionRef, {
                name,
                faculty,
                year,
                accidentType,
                location,
                description,
                severity,
                injuries: parseInt(injuries),
                remarks,
                timestamp: serverTimestamp()
            });
            showUserMessage("รายงานอุบัติเหตุสำเร็จ!");
            // Clear form fields
            setName('');
            setFaculty('คณะเกษตร กำแพงแสน');
            setYear('ปีที่ 1');
            setAccidentType('อุบัติเหตุจราจร');
            setLocation('');
            setDescription('');
            setSeverity('เล็กน้อย');
            setInjuries(0);
            setRemarks('');
        } catch (error) {
            console.error("Error adding document:", error);
            showUserMessage("เกิดข้อผิดพลาดในการบันทึกข้อมูล: " + error.message);
        }
    };

    // Handle file selection for import
    const handleFileChange = (event) => {
        setSelectedFile(event.target.files[0]);
    };

    // Function to parse CSV and import data
    const handleImportCSV = async () => {
        if (!selectedFile) {
            showUserMessage("กรุณาเลือกไฟล์ CSV ที่ต้องการนำเข้า");
            return;
        }
        if (selectedFile.type !== 'text/csv') {
            showUserMessage("กรุณาเลือกไฟล์ CSV เท่านั้น (.csv)");
            return;
        }

        setIsImporting(true);
        const reader = new FileReader();

        reader.onload = async (e) => {
            const text = e.target.result;
            const lines = text.split('\n').filter(line => line.trim() !== '');
            if (lines.length === 0) {
                showUserMessage("ไฟล์ CSV ว่างเปล่า");
                setIsImporting(false);
                return;
            }

            const headers = lines[0].split(',').map(header => header.trim().replace(/"/g, ''));
            const dataToImport = [];

            for (let i = 1; i < lines.length; i++) {
                const values = lines[i].split(/,(?=(?:(?:[^"]*"){2})*[^"]*$)/).map(value => value.trim().replace(/"/g, ''));
                if (values.length !== headers.length) {
                    console.warn(`Skipping line ${i + 1} due to mismatched column count.`);
                    continue;
                }

                const rowData = {};
                headers.forEach((header, index) => {
                    switch (header) {
                        case 'ชื่อ': rowData.name = values[index]; break;
                        case 'คณะ/หน่วยงาน': rowData.faculty = values[index]; break;
                        case 'ชั้นปี/ตำแหน่ง': rowData.year = values[index]; break;
                        case 'ประเภทอุบัติเหตุ': rowData.accidentType = values[index]; break;
                        case 'สถานที่เกิดเหตุ': rowData.location = values[index]; break;
                        case 'รายละเอียด': rowData.description = values[index]; break;
                        case 'ความรุนแรง': rowData.severity = values[index]; break;
                        case 'จำนวนผู้บาดเจ็บ': rowData.injuries = parseInt(values[index]) || 0; break;
                        case 'หมายเหตุ': rowData.remarks = values[index]; break;
                        default: break;
                    }
                });
                if (rowData.name && rowData.faculty && rowData.year && rowData.location && rowData.description) {
                    dataToImport.push(rowData);
                } else {
                    console.warn(`Skipping row due to missing required fields: ${JSON.stringify(rowData)}`);
                }
            }

            if (dataToImport.length === 0) {
                showUserMessage("ไม่มีข้อมูลที่ถูกต้องจากไฟล์ CSV ที่จะนำเข้าได้");
                setIsImporting(false);
                return;
            }

            try {
                const accidentsCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/accidents`);
                let importedCount = 0;
                for (const data of dataToImport) {
                    await addDoc(accidentsCollectionRef, {
                        ...data,
                        timestamp: serverTimestamp()
                    });
                    importedCount++;
                }
                showUserMessage(`นำเข้าข้อมูลสำเร็จ ${importedCount} รายการ!`);
                setSelectedFile(null); // Clear selected file
            } catch (error) {
                console.error("Error importing data:", error);
                showUserMessage("เกิดข้อผิดพลาดในการนำเข้าข้อมูล: " + error.message);
            } finally {
                setIsImporting(false);
            }
        };

        reader.onerror = () => {
            showUserMessage("เกิดข้อผิดพลาดในการอ่านไฟล์");
            setIsImporting(false);
        };

        reader.readAsText(selectedFile);
    };

    // Filter accidents based on selected date range
    const filteredAccidents = accidents.filter(accident => {
        if (!startDate && !endDate) return true; // No filter applied

        const accidentDate = accident.timestamp ? accident.timestamp.toDate() : null;
        if (!accidentDate) return false; // Skip if no timestamp

        let start = startDate ? new Date(startDate) : null;
        let end = endDate ? new Date(endDate) : null;

        // Set end date to the end of the day for inclusive range
        if (end) {
            end.setHours(23, 59, 59, 999);
        }

        if (start && end) {
            return accidentDate >= start && accidentDate <= end;
        } else if (start) {
            return accidentDate >= start;
        } else if (end) {
            return accidentDate <= end;
        }
        return true;
    });

    // Calculate summary statistics based on filtered accidents
    const totalAccidents = filteredAccidents.length;
    const totalInjuries = filteredAccidents.reduce((sum, accident) => sum + (accident.injuries || 0), 0);
    
    // Summary by Severity
    const severityCounts = filteredAccidents.reduce((counts, accident) => {
        const currentSeverity = accident.severity || 'ไม่ระบุ';
        counts[currentSeverity] = (counts[currentSeverity] || 0) + 1;
        return counts;
    }, {});

    // Summary by Accident Type
    const accidentTypeCounts = filteredAccidents.reduce((counts, accident) => {
        const type = accident.accidentType || 'ไม่ระบุ';
        counts[type] = (counts[type] || 0) + 1;
        return counts;
    }, {});

    // Summary by Year Level
    const yearLevelCounts = filteredAccidents.reduce((counts, accident) => {
        const level = accident.year || 'ไม่ระบุ';
        counts[level] = (counts[level] || 0) + 1;
        return counts;
    }, {});

    // Summary by Faculty
    const facultyCounts = filteredAccidents.reduce((counts, accident) => {
        const fac = accident.faculty || 'ไม่ระบุ';
        counts[fac] = (counts[fac] || 0) + 1;
        return counts;
    }, {});

    // Summary by Location
    const locationCounts = filteredAccidents.reduce((counts, accident) => {
        const loc = accident.location || 'ไม่ระบุ';
        counts[loc] = (counts[loc] || 0) + 1;
        return counts;
    }, {});

    // --- Download Functions ---

    // Function to download all accident data as CSV
    const downloadAccidentsCSV = () => {
        if (filteredAccidents.length === 0) {
            showUserMessage("ไม่มีข้อมูลอุบัติเหตุในขอบเขตวันที่ที่เลือกให้ดาวน์โหลด");
            return;
        }

        // Define CSV headers
        const headers = [
            "วันที่/เวลา", "ชื่อ", "คณะ/หน่วยงาน", "ชั้นปี/ตำแหน่ง", 
            "ประเภทอุบัติเหตุ", "สถานที่เกิดเหตุ", "รายละเอียด", 
            "ความรุนแรง", "จำนวนผู้บาดเจ็บ", "หมายเหตุ"
        ];

        // Map accident data to CSV rows
        const csvRows = filteredAccidents.map(accident => {
            const date = accident.timestamp ? new Date(accident.timestamp.toDate()).toLocaleString('th-TH') : 'N/A';
            return [
                `"${date}"`,
                `"${accident.name || ''}"`,
                `"${accident.faculty || ''}"`,
                `"${accident.year || ''}"`,
                `"${accident.accidentType || ''}"`,
                `"${accident.location || ''}"`,
                `"${accident.description || ''}"`,
                `"${accident.severity || ''}"`,
                `"${accident.injuries || 0}"`,
                `"${accident.remarks || ''}"`
            ].join(',');
        });

        const csvContent = [
            headers.join(','),
            ...csvRows
        ].join('\n');

        const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' });
        const link = document.createElement('a');
        if (link.download !== undefined) { // feature detection
            const url = URL.createObjectURL(blob);
            link.setAttribute('href', url);
            link.setAttribute('download', `accident_data_filtered_${new Date().toISOString().slice(0,10)}.csv`);
            link.style.visibility = 'hidden';
            document.body.appendChild(link);
            link.click();
            document.body.removeChild(link);
        }
        showUserMessage("ดาวน์โหลดข้อมูลอุบัติเหตุเสร็จสมบูรณ์");
    };

    // Function to download summary report as PDF (mockup)
    const downloadSummaryPDF = () => {
        if (filteredAccidents.length === 0) {
            showUserMessage("ไม่มีข้อมูลอุบัติเหตุในขอบเขตวันที่ที่เลือกสำหรับรายงานสรุป");
            return;
        }

        const summaryText = `
            รายงานสรุปอุบัติเหตุ สถานพยาบาลมหาวิทยาลัยเกษตรศาสตร์ วิทยาเขตกำแพงแสน
            (ข้อมูล ณ วันที่ ${new Date().toLocaleDateString('th-TH')})
            ${startDate && endDate ? `ช่วงวันที่: ${new Date(startDate).toLocaleDateString('th-TH')} ถึง ${new Date(endDate).toLocaleDateString('th-TH')}` : 'ทุกช่วงเวลา'}

            --------------------------------------------------------
            สรุปภาพรวม:
            จำนวนอุบัติเหตุทั้งหมด: ${totalAccidents} ครั้ง
            จำนวนผู้บาดเจ็บทั้งหมด: ${totalInjuries} คน

            --------------------------------------------------------
            สรุปตามความรุนแรง:
            ${Object.keys(severityCounts).length > 0 ?
                Object.entries(severityCounts).map(([key, value]) => `- ${key}: ${value} ครั้ง`).join('\n')
                : 'ยังไม่มีข้อมูลความรุนแรง'}

            --------------------------------------------------------
            สรุปตามประเภทอุบัติเหตุ:
            ${Object.keys(accidentTypeCounts).length > 0 ?
                Object.entries(accidentTypeCounts).map(([type, count]) => `- ${type}: ${count} ครั้ง`).join('\n')
                : 'ยังไม่มีข้อมูลประเภทอุบัติเหตุ'}

            --------------------------------------------------------
            สรุปตามชั้นปี/ตำแหน่ง:
            ${Object.keys(yearLevelCounts).length > 0 ?
                Object.entries(yearLevelCounts).map(([level, count]) => `- ${level}: ${count} ครั้ง`).join('\n')
                : 'ยังไม่มีข้อมูลชั้นปี/ตำแหน่ง'}

            --------------------------------------------------------
            สรุปตามคณะ/หน่วยงาน:
            ${Object.keys(facultyCounts).length > 0 ?
                Object.entries(facultyCounts).map(([fac, count]) => `- ${fac}: ${count} ครั้ง`).join('\n')
                : 'ยังไม่มีข้อมูลคณะ/หน่วยงาน'}

            --------------------------------------------------------
            สรุปตามสถานที่เกิดเหตุ:
            ${Object.keys(locationCounts).length > 0 ?
                Object.entries(locationCounts).map(([loc, count]) => `- ${loc}: ${count} ครั้ง`).join('\n')
                : 'ยังไม่มีข้อมูลสถานที่เกิดเหตุ'}

            --------------------------------------------------------
            ข้อมูลจากระบบรายงานอุบัติเหตุ
            Generated by Accident Reporting App
        `;

        const blob = new Blob([summaryText], { type: 'text/plain;charset=utf-8;' });
        const link = document.createElement('a');
        if (link.download !== undefined) {
            const url = URL.createObjectURL(blob);
            link.setAttribute('href', url);
            link.setAttribute('download', `accident_summary_report_filtered_${new Date().toISOString().slice(0,10)}.txt`);
            link.style.visibility = 'hidden';
            document.body.appendChild(link);
            link.click();
            document.body.removeChild(link);
        }
        showUserMessage("ดาวน์โหลดรายงานสรุปเสร็จสมบูรณ์ (เป็นไฟล์ข้อความตัวอย่าง)");
    };

    // --- Gemini AI Integration ---
    const analyzeAccidentWithGemini = async () => {
        if (!aiAnalysisInput.trim()) {
            showUserMessage("กรุณาป้อนรายละเอียดอุบัติเหตุเพื่อวิเคราะห์");
            return;
        }

        setAiAnalysisOutput(''); // Clear previous output
        setIsAiLoading(true); // Show loading indicator

        const prompt = `คุณเป็นผู้เชี่ยวชาญด้านการวิเคราะห์อุบัติเหตุ โปรดวิเคราะห์รายละเอียดอุบัติเหตุต่อไปนี้และให้ข้อเสนอแนะเพื่อป้องกันไม่ให้เกิดซ้ำในอนาคต โดยเน้นที่สาเหตุหลักและมาตรการป้องกันที่ปฏิบัติได้จริง:
        
        รายละเอียดอุบัติเหตุ:
        ${aiAnalysisInput}
        
        ผลการวิเคราะห์และข้อเสนอแนะ:`;

        let chatHistory = [];
        chatHistory.push({ role: "user", parts: [{ text: prompt }] });

        const payload = { contents: chatHistory };
        const apiKey = ""; // Leave this as-is; Canvas will provide it at runtime.
        const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=${apiKey}`;

        try {
            const response = await fetch(apiUrl, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(payload)
            });

            if (!response.ok) {
                const errorData = await response.json();
                throw new Error(`API error: ${response.status} ${response.statusText} - ${errorData.error?.message || 'Unknown error'}`);
            }

            const result = await response.json();
            if (result.candidates && result.candidates.length > 0 &&
                result.candidates[0].content && result.candidates[0].content.parts &&
                result.candidates[0].content.parts.length > 0) {
                const text = result.candidates[0].content.parts[0].text;
                setAiAnalysisOutput(text); // Display the AI response
            } else {
                setAiAnalysisOutput("ไม่สามารถวิเคราะห์ได้: ไม่ได้รับคำตอบที่ถูกต้องจาก AI.");
                console.error("Unexpected API response structure:", result);
            }
        } catch (error) {
            console.error("Error calling Gemini API:", error);
            setAiAnalysisOutput(`เกิดข้อผิดพลาดในการวิเคราะห์ด้วย AI: ${error.message}`);
        } finally {
            setIsAiLoading(false); // Hide loading indicator
        }
    };

    return (
        <div className="min-h-screen bg-gradient-to-br from-blue-100 to-indigo-200 p-4 font-sans text-gray-800 flex flex-col items-center">
            <script src="https://cdn.tailwindcss.com"></script>
            <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet" />
            <style>
                {`
                body {
                    font-family: 'Inter', sans-serif;
                }
                .modal {
                    transition: opacity 0.3s ease-in-out;
                }
                .modal.hidden {
                    opacity: 0;
                    pointer-events: none;
                }
                /* Custom styling for file input to match Tailwind aesthetics */
                .file-input-wrapper input[type="file"] {
                    display: none; /* Hide default file input */
                }
                .file-input-wrapper label {
                    display: inline-block;
                    cursor: pointer;
                    padding: 0.75rem 1.5rem;
                    border-radius: 0.375rem; /* rounded-md */
                    border: 1px solid #d1d5db; /* border-gray-300 */
                    font-weight: 600; /* font-semibold */
                    font-size: 0.875rem; /* text-sm */
                    background-color: #eff6ff; /* bg-blue-50 */
                    color: #1d4ed8; /* text-blue-700 */
                    transition: background-color 0.2s ease-in-out;
                }
                .file-input-wrapper label:hover {
                    background-color: #bfdbfe; /* hover:bg-blue-100 */
                }
                `}
            </style>

            {/* User ID Display */}
            {userId && (
                <div className="bg-white p-3 rounded-lg shadow-md mb-6 w-full max-w-4xl text-center text-sm">
                    <p className="font-semibold">รหัสผู้ใช้ของคุณ: <span className="font-normal break-all">{userId}</span></p>
                    <p className="text-xs text-gray-500 mt-1">ใช้รหัสนี้เพื่อระบุตัวตนของคุณในระบบ</p>
                </div>
            )}

            {/* Modal for messages */}
            <div className={`fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center z-50 modal ${showModal ? '' : 'hidden'}`}>
                <div className="bg-white p-6 rounded-lg shadow-xl max-w-sm w-full text-center">
                    <h3 className="text-lg font-semibold mb-4">ข้อความ</h3>
                    <p className="mb-6">{modalMessage}</p>
                    <button
                        onClick={() => setShowModal(false)}
                        className="bg-blue-600 text-white px-6 py-2 rounded-md hover:bg-blue-700 transition duration-200 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-opacity-50"
                    >
                        ตกลง
                    </button>
                </div>
            </div>

            {/* Updated H1 title */}
            <h1 className="text-3xl md:text-4xl font-bold text-blue-800 mb-8 drop-shadow-lg text-center leading-tight">
                ระบบรายงานอุบัติเหตุ <br /> สถานพยาบาลมหาวิทยาลัยเกษตรศาสตร์ วิทยาเขตกำแพงแสน
            </h1>

            {/* Accident Reporting Form */}
            <div className="bg-white p-8 rounded-xl shadow-2xl mb-8 w-full max-w-4xl border border-blue-200">
                <h2 className="text-2xl font-semibold text-blue-700 mb-6 border-b pb-3">รายงานอุบัติเหตุใหม่</h2>
                <form onSubmit={handleSubmit} className="space-y-5">
                    {/* Name, Faculty, Year/Position */}
                    <div className="grid grid-cols-1 md:grid-cols-3 gap-5">
                        <div>
                            <label htmlFor="name" className="block text-sm font-medium text-gray-700 mb-1">ชื่อ-นามสกุล:</label>
                            <input
                                type="text"
                                id="name"
                                value={name}
                                onChange={(e) => setName(e.target.value)}
                                className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-blue-500 focus:border-blue-500 sm:text-sm transition duration-150 ease-in-out"
                                placeholder="ชื่อ-นามสกุล"
                                required
                            />
                        </div>
                        <div>
                            <label htmlFor="faculty" className="block text-sm font-medium text-gray-700 mb-1">คณะ/หน่วยงาน:</label>
                            <select
                                id="faculty"
                                value={faculty}
                                onChange={(e) => setFaculty(e.target.value)}
                                className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-blue-500 focus:border-blue-500 sm:text-sm transition duration-150 ease-in-out"
                                required
                            >
                                {faculties.map((fac, index) => (
                                    <option key={index} value={fac}>{fac}</option>
                                ))}
                            </select>
                        </div>
                        <div>
                            <label htmlFor="year" className="block text-sm font-medium text-gray-700 mb-1">ชั้นปี/ตำแหน่ง:</label>
                            <select
                                id="year"
                                value={year}
                                onChange={(e) => setYear(e.target.value)}
                                className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-blue-500 focus:border-blue-500 sm:text-sm transition duration-150 ease-in-out"
                                required
                            >
                                {yearLevels.map((level, index) => (
                                    <option key={index} value={level}>{level}</option>
                                ))}
                            </select>
                        </div>
                    </div>

                    {/* Accident Type */}
                    <div>
                        <label htmlFor="accidentType" className="block text-sm font-medium text-gray-700 mb-1">ประเภทอุบัติเหตุ:</label>
                        <select
                            id="accidentType"
                            value={accidentType}
                            onChange={(e) => setAccidentType(e.target.value)}
                            className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-blue-500 focus:border-blue-500 sm:text-sm transition duration-150 ease-in-out"
                        >
                            <option value="อุบัติเหตุจราจร">อุบัติเหตุจราจร</option>
                            <option value="อุบัติเหตุทั่วไป">อุบัติเหตุทั่วไป</option>
                            <option value="อุบัติเหตุจากการทำงาน">อุบัติเหตุจากการทำงาน</option>
                            <option value="อุบัติเหตุการเล่นกีฬา">อุบัติเหตุการเล่นกีฬา</option>
                            <option value="อุบัติเหตุจากรถจักรยานไฟฟ้า">อุบัติเหตุจากรถจักรยานไฟฟ้า</option>
                        </select>
                    </div>

                    {/* Location and Description */}
                    <div>
                        <label htmlFor="location" className="block text-sm font-medium text-gray-700 mb-1">สถานที่เกิดเหตุ:</label>
                        <input
                            type="text"
                            id="location"
                            value={location}
                            onChange={(e) => setLocation(e.target.value)}
                            className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-blue-500 focus:border-blue-500 sm:text-sm transition duration-150 ease-in-out"
                            placeholder="เช่น ถนนสุขุมวิท, โรงงาน A"
                            required
                        />
                    </div>
                    <div>
                        <label htmlFor="description" className="block text-sm font-medium text-gray-700 mb-1">รายละเอียด:</label>
                        <textarea
                            id="description"
                            value={description}
                            onChange={(e) => setDescription(e.target.value)}
                            rows="4"
                            className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-blue-500 focus:border-blue-500 sm:text-sm transition duration-150 ease-in-out"
                            placeholder="อธิบายเหตุการณ์โดยย่อ"
                            required
                        ></textarea>
                    </div>

                    {/* Severity and Injuries */}
                    <div className="grid grid-cols-1 md:grid-cols-2 gap-5">
                        <div>
                            <label htmlFor="severity" className="block text-sm font-medium text-gray-700 mb-1">ความรุนแรง:</label>
                            <select
                                id="severity"
                                value={severity}
                                onChange={(e) => setSeverity(e.target.value)}
                                className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-blue-500 focus:border-blue-500 sm:text-sm transition duration-150 ease-in-out"
                            >
                                <option value="เล็กน้อย">เล็กน้อย</option>
                                <option value="ปานกลาง">ปานกลาง</option>
                                <option value="รุนแรง">รุนแรง</option>
                                <option value="ร้ายแรงมาก">ร้ายแรงมาก</option>
                            </select>
                        </div>
                        <div>
                            <label htmlFor="injuries" className="block text-sm font-medium text-gray-700 mb-1">จำนวนผู้บาดเจ็บ:</label>
                            <input
                                type="number"
                                id="injuries"
                                value={injuries}
                                onChange={(e) => setInjuries(Math.max(0, parseInt(e.target.value) || 0))} // Ensure non-negative
                                min="0"
                                className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-blue-500 focus:border-blue-500 sm:text-sm transition duration-150 ease-in-out"
                            />
                        </div>
                    </div>

                    {/* New field: Remarks */}
                    <div>
                        <label htmlFor="remarks" className="block text-sm font-medium text-gray-700 mb-1">หมายเหตุ (ถ้ามี):</label>
                        <textarea
                            id="remarks"
                            value={remarks}
                            onChange={(e) => setRemarks(e.target.value)}
                            rows="2"
                            className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-blue-500 focus:border-blue-500 sm:text-sm transition duration-150 ease-in-out"
                            placeholder="ข้อมูลเพิ่มเติมเกี่ยวกับอุบัติเหตุ"
                        ></textarea>
                    </div>

                    <button
                        type="submit"
                        className="w-full bg-blue-600 text-white py-3 px-4 rounded-md hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2 transition duration-200 ease-in-out text-lg font-semibold shadow-lg transform hover:scale-105"
                    >
                        บันทึกรายงาน
                    </button>
                </form>
            </div>

            {/* Import Data Section */}
            <div className="bg-white p-8 rounded-xl shadow-2xl mb-8 w-full max-w-4xl border border-yellow-200">
                <h2 className="text-2xl font-semibold text-yellow-700 mb-6 border-b pb-3">นำเข้าข้อมูลจากไฟล์ (CSV)</h2>
                <div className="flex flex-col md:flex-row items-center justify-center gap-4">
                    <div className="file-input-wrapper">
                        <input type="file" id="fileInput" accept=".csv" onChange={handleFileChange} />
                        <label htmlFor="fileInput">{selectedFile ? selectedFile.name : 'เลือกไฟล์ CSV'}</label>
                    </div>
                    <button
                        onClick={handleImportCSV}
                        disabled={!selectedFile || isImporting}
                        className={`py-3 px-6 rounded-md text-lg font-semibold shadow-lg transform hover:scale-105
                                   ${!selectedFile || isImporting ? 'bg-gray-400 cursor-not-allowed' : 'bg-green-600 text-white hover:bg-green-700 focus:outline-none focus:ring-2 focus:ring-green-500 focus:ring-offset-2 transition duration-200 ease-in-out'}`}
                    >
                        {isImporting ? 'กำลังนำเข้า...' : 'นำเข้าข้อมูล CSV'}
                    </button>
                </div>
                <p className="text-sm text-gray-500 mt-4 text-center">
                    *รองรับเฉพาะไฟล์ CSV (.csv) เท่านั้น. สำหรับไฟล์ Excel (.xlsx) ต้องใช้ไลบรารีเพิ่มเติม.
                    <br/>
                    <span className="font-semibold">รูปแบบหัวข้อ CSV ที่รองรับ:</span> "วันที่/เวลา", "ชื่อ", "คณะ/หน่วยงาน", "ชั้นปี/ตำแหน่ง", "ประเภทอุบัติเหตุ", "สถานที่เกิดเหตุ", "รายละเอียด", "ความรุนแรง", "จำนวนผู้บาดเจ็บ", "หมายเหตุ"
                </p>
            </div>

            {/* Gemini AI Analysis Section */}
            <div className="bg-white p-8 rounded-xl shadow-2xl mb-8 w-full max-w-4xl border border-pink-200">
                <h2 className="text-2xl font-semibold text-pink-700 mb-6 border-b pb-3">✨ วิเคราะห์และเสนอแนะอุบัติเหตุด้วย AI ✨</h2>
                <div className="space-y-4">
                    <div>
                        <label htmlFor="aiAnalysisInput" className="block text-sm font-medium text-gray-700 mb-1">ป้อนรายละเอียดอุบัติเหตุเพื่อวิเคราะห์:</label>
                        <textarea
                            id="aiAnalysisInput"
                            rows="6"
                            value={aiAnalysisInput}
                            onChange={(e) => setAiAnalysisInput(e.target.value)}
                            className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-pink-500 focus:border-pink-500 sm:text-sm transition duration-150 ease-in-out"
                            placeholder="คัดลอกรายละเอียดอุบัติเหตุจากรายการด้านล่าง หรือป้อนข้อมูลใหม่"
                        ></textarea>
                    </div>
                    <button
                        onClick={analyzeAccidentWithGemini}
                        disabled={isAiLoading}
                        className={`w-full py-3 px-4 rounded-md text-lg font-semibold shadow-lg transform hover:scale-105
                                   ${isAiLoading ? 'bg-gray-400 cursor-not-allowed' : 'bg-pink-600 text-white hover:bg-pink-700 focus:outline-none focus:ring-2 focus:ring-pink-500 focus:ring-offset-2 transition duration-200 ease-in-out'}`}
                    >
                        {isAiLoading ? 'กำลังวิเคราะห์...' : 'วิเคราะห์และเสนอแนะด้วย AI'}
                    </button>
                    <div className="mt-4 p-4 bg-pink-50 rounded-lg border border-pink-100 text-gray-800 whitespace-pre-wrap">
                        {aiAnalysisOutput}
                    </div>
                </div>
            </div>

            {/* Summary Section */}
            <div className="bg-white p-8 rounded-xl shadow-2xl mb-8 w-full max-w-4xl border border-green-200">
                <h2 className="text-2xl font-semibold text-green-700 mb-6 border-b pb-3">สรุปข้อมูลอุบัติเหตุ</h2>
                <div className="grid grid-cols-1 md:grid-cols-3 gap-6 text-center">
                    <div className="bg-green-50 p-6 rounded-lg shadow-md border border-green-100">
                        <p className="text-4xl font-bold text-green-800">{totalAccidents}</p>
                        <p className="text-lg text-green-600 mt-2">อุบัติเหตุทั้งหมด</p>
                    </div>
                    <div className="bg-green-50 p-6 rounded-lg shadow-md border border-green-100">
                        <p className="text-4xl font-bold text-green-800">{totalInjuries}</p>
                        <p className="text-lg text-green-600 mt-2">ผู้บาดเจ็บทั้งหมด</p>
                    </div>
                    <div className="bg-green-50 p-6 rounded-lg shadow-md border border-green-100">
                        <p className="text-lg font-semibold text-green-700 mb-3">แยกตามความรุนแรง:</p>
                        {Object.keys(severityCounts).length > 0 ? (
                            <ul className="text-left inline-block">
                                {Object.entries(severityCounts).map(([key, value]) => (
                                    <li key={key} className="text-md text-gray-700">
                                        <span className="font-medium">{key}:</span> {value} ครั้ง
                                    </li>
                                ))}
                            </ul>
                        ) : (
                            <p className="text-gray-500 text-sm">ยังไม่มีข้อมูล</p>
                        )}
                    </div>
                </div>
                
                {/* New Summary Sections */}
                <div className="mt-8 grid grid-cols-1 md:grid-cols-2 gap-6">
                    <div>
                        <h3 className="text-xl font-semibold text-green-700 mb-4 border-t pt-4">สรุปตามประเภทอุบัติเหตุ:</h3>
                        {Object.keys(accidentTypeCounts).length > 0 ? (
                            <ul className="space-y-2">
                                {Object.entries(accidentTypeCounts).map(([type, count]) => (
                                    <li key={type} className="bg-green-50 p-3 rounded-lg shadow-sm border border-green-100 flex justify-between items-center">
                                        <span className="font-medium text-green-800">{type}</span>
                                        <span className="text-lg font-bold text-green-700">{count} ครั้ง</span>
                                    </li>
                                ))}
                            </ul>
                        ) : (
                            <p className="text-gray-500 text-center text-sm">ยังไม่มีข้อมูลประเภทอุบัติเหตุ</p>
                        )}
                    </div>
                    <div>
                        <h3 className="text-xl font-semibold text-green-700 mb-4 border-t pt-4">สรุปตามชั้นปี/ตำแหน่ง:</h3>
                        {Object.keys(yearLevelCounts).length > 0 ? (
                            <ul className="space-y-2">
                                {Object.entries(yearLevelCounts).map(([level, count]) => (
                                    <li key={level} className="bg-green-50 p-3 rounded-lg shadow-sm border border-green-100 flex justify-between items-center">
                                        <span className="font-medium text-green-800">{level}</span>
                                        <span className="text-lg font-bold text-green-700">{count} ครั้ง</span>
                                    </li>
                                ))}
                            </ul>
                        ) : (
                            <p className="text-gray-500 text-center text-sm">ยังไม่มีข้อมูลชั้นปี/ตำแหน่ง</p>
                        )}
                    </div>
                </div>

                <div className="mt-8 grid grid-cols-1 md:grid-cols-2 gap-6">
                    <div>
                        <h3 className="text-xl font-semibold text-green-700 mb-4 border-t pt-4">สรุปตามคณะ/หน่วยงาน:</h3>
                        {Object.keys(facultyCounts).length > 0 ? (
                            <ul className="space-y-2">
                                {Object.entries(facultyCounts).map(([fac, count]) => (
                                    <li key={fac} className="bg-green-50 p-3 rounded-lg shadow-sm border border-green-100 flex justify-between items-center">
                                        <span className="font-medium text-green-800">{fac}</span>
                                        <span className="text-lg font-bold text-green-700">{count} ครั้ง</span>
                                    </li>
                                ))}
                            </ul>
                        ) : (
                            <p className="text-gray-500 text-center text-sm">ยังไม่มีข้อมูลคณะ/หน่วยงาน</p>
                        )}
                    </div>
                    <div>
                        <h3 className="text-xl font-semibold text-green-700 mb-4 border-t pt-4">สรุปตามสถานที่เกิดเหตุ:</h3>
                        {Object.keys(locationCounts).length > 0 ? (
                            <ul className="space-y-2">
                                {Object.entries(locationCounts).map(([loc, count]) => (
                                    <li key={loc} className="bg-green-50 p-3 rounded-lg shadow-sm border border-green-100 flex justify-between items-center">
                                        <span className="font-medium text-green-800">{loc}</span>
                                        <span className="text-lg font-bold text-green-700">{count} ครั้ง</span>
                                    </li>
                                ))}
                            </ul>
                        ) : (
                            <p className="text-gray-500 text-center text-sm">ยังไม่มีข้อมูลสถานที่เกิดเหตุ</p>
                        )}
                    </div>
                </div>
            </div>

            {/* Download Buttons Section */}
            <div className="bg-white p-8 rounded-xl shadow-2xl mb-8 w-full max-w-4xl border border-teal-200 text-center">
                <h2 className="text-2xl font-semibold text-teal-700 mb-6 border-b pb-3">ดาวน์โหลดข้อมูลและรายงาน</h2>
                <div className="flex flex-col md:flex-row justify-center items-center gap-4 mb-6">
                    <div>
                        <label htmlFor="startDate" className="block text-sm font-medium text-gray-700 mb-1">วันที่เริ่มต้น:</label>
                        <input
                            type="date"
                            id="startDate"
                            value={startDate}
                            onChange={(e) => setStartDate(e.target.value)}
                            className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-blue-500 focus:border-blue-500 sm:text-sm transition duration-150 ease-in-out"
                        />
                    </div>
                    <div>
                        <label htmlFor="endDate" className="block text-sm font-medium text-gray-700 mb-1">วันที่สิ้นสุด:</label>
                        <input
                            type="date"
                            id="endDate"
                            value={endDate}
                            onChange={(e) => setEndDate(e.target.value)}
                            className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-blue-500 focus:border-blue-500 sm:text-sm transition duration-150 ease-in-out"
                        />
                    </div>
                </div>
                <div className="flex flex-col md:flex-row justify-center gap-4">
                    <button
                        onClick={downloadAccidentsCSV}
                        className="bg-purple-600 text-white py-3 px-6 rounded-md hover:bg-purple-700 focus:outline-none focus:ring-2 focus:ring-purple-500 focus:ring-offset-2 transition duration-200 ease-in-out text-lg font-semibold shadow-lg transform hover:scale-105"
                    >
                        ดาวน์โหลดข้อมูลทั้งหมด (CSV)
                    </button>
                    <button
                        onClick={downloadSummaryPDF}
                        className="bg-orange-600 text-white py-3 px-6 rounded-md hover:bg-orange-700 focus:outline-none focus:ring-2 focus:ring-orange-500 focus:ring-offset-2 transition duration-200 ease-in-out text-lg font-semibold shadow-lg transform hover:scale-105"
                    >
                        ดาวน์โหลดรายงานสรุป (PDF ตัวอย่าง)
                    </button>
                </div>
            </div>

            {/* Accident List */}
            <div className="bg-white p-8 rounded-xl shadow-2xl w-full max-w-4xl border border-purple-200">
                <h2 className="text-2xl font-semibold text-purple-700 mb-6 border-b pb-3">รายการอุบัติเหตุที่บันทึกไว้</h2>
                {filteredAccidents.length === 0 ? (
                    <p className="text-center text-gray-500 py-8">
                        ยังไม่มีอุบัติเหตุที่บันทึกไว้
                        {startDate || endDate ? 'ในขอบเขตวันที่ที่เลือก' : ''}
                    </p>
                ) : (
                    <div className="overflow-x-auto">
                        <table className="min-w-full divide-y divide-gray-200">
                            <thead className="bg-gray-50">
                                <tr>
                                    <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider rounded-tl-lg">
                                        วันที่/เวลา
                                    </th>
                                    <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                                        ชื่อ
                                    </th>
                                    <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                                        คณะ/หน่วยงาน
                                    </th>
                                    <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                                        ชั้นปี/ตำแหน่ง
                                    </th>
                                    <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                                        ประเภทอุบัติเหตุ
                                    </th>
                                    <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                                        สถานที่
                                    </th>
                                    <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                                        รายละเอียด
                                    </th>
                                    <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                                        ความรุนแรง
                                    </th>
                                    <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                                        ผู้บาดเจ็บ
                                    </th>
                                    <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider rounded-tr-lg">
                                        หมายเหตุ
                                    </th>
                                </tr>
                            </thead>
                            <tbody className="bg-white divide-y divide-gray-200">
                                {filteredAccidents.map((accident) => (
                                    <tr key={accident.id} className="hover:bg-gray-50">
                                        <td className="px-6 py-4 whitespace-nowrap text-sm font-medium text-gray-900">
                                            {accident.timestamp ? new Date(accident.timestamp.toDate()).toLocaleString('th-TH') : 'N/A'}
                                        </td>
                                        <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-700">
                                            {accident.name}
                                        </td>
                                        <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-700">
                                            {accident.faculty}
                                        </td>
                                        <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-700">
                                            {accident.year}
                                        </td>
                                        <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-700">
                                            {accident.accidentType}
                                        </td>
                                        <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-700">
                                            {accident.location}
                                        </td>
                                        <td className="px-6 py-4 text-sm text-gray-700 max-w-xs truncate">
                                            {accident.description}
                                        </td>
                                        <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-700">
                                            {accident.severity}
                                        </td>
                                        <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-700">
                                            {accident.injuries}
                                        </td>
                                        <td className="px-6 py-4 text-sm text-gray-700 max-w-xs truncate">
                                            {accident.remarks || '-'}
                                        </td>
                                    </tr>
                                ))}
                            </tbody>
                        </table>
                    </div>
                )}
            </div>
        </div>
    );
};

export default App;

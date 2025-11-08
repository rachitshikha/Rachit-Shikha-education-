/*
Rachitshikha - React PWA (Firebase-connected App.jsx)
Updated: Firebase Authentication + Firestore wiring (email/password), persistent Notes, Jobs, Quizzes, Profile and simple earnings flow.

How to use:
1. Create React app:
   npx create-react-app rachitshikha
   cd rachitshikha
2. Install deps:
   npm install firebase react-router-dom
3. Replace src/App.jsx with this file content.
4. Replace src/index.css with your preferred CSS or Tailwind setup.
5. Create a Firebase project at https://console.firebase.google.com/
   - Enable Authentication -> Email/Password
   - Enable Firestore Database (start in test mode for development)
6. Copy your Firebase config and paste into the firebaseConfig object below.
7. Run:
   npm start
8. Deploy: Use Vercel/Netlify/GitHub Pages. To make installable on Android, configure as PWA and 'Add to Home screen' from Chrome.

Notes:
- This is still a simplified example. Add security rules and server-side verification for production.
- For payments/withdrawals integrate Razorpay/Stripe and a secure backend.
*/

import React, { useState, useEffect } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, onAuthStateChanged, signInWithEmailAndPassword, createUserWithEmailAndPassword, signOut } from 'firebase/auth';
import { getFirestore, collection, getDocs, addDoc, query, where, doc, getDoc, updateDoc, orderBy } from 'firebase/firestore';

// ---------- FIREBASE CONFIG - REPLACE WITH YOURS ----------
const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_AUTH_DOMAIN",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_STORAGE_BUCKET",
  messagingSenderId: "YOUR_MESSAGING_SENDER_ID",
  appId: "YOUR_APP_ID"
};

const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);

// ---------- APP ----------
export default function App() {
  const [route, setRoute] = useState('home');
  const [user, setUser] = useState(null); // Firebase user object
  const [profile, setProfile] = useState(null);
  const [notes, setNotes] = useState([]);
  const [jobs, setJobs] = useState([]);
  const [quizzes, setQuizzes] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Auth state listener
    const unsub = onAuthStateChanged(auth, async (u) => {
      if (u) {
        setUser(u);
        await loadProfile(u.uid);
      } else {
        setUser(null);
        setProfile(null);
      }
      setLoading(false);
    });

    // Load public content
    loadNotes();
    loadJobs();
    loadQuizzes();

    return () => unsub();
  }, []);

  // ---------- FIRESTORE READS ----------
  async function loadNotes(){
    const snap = await getDocs(collection(db, 'notes'));
    setNotes(snap.docs.map(d=>({ id: d.id, ...d.data() })));
  }

  async function loadJobs(){
    const snap = await getDocs(collection(db, 'jobs'));
    setJobs(snap.docs.map(d=>({ id: d.id, ...d.data() })));
  }

  async function loadQuizzes(){
    const snap = await getDocs(collection(db, 'quizzes'));
    setQuizzes(snap.docs.map(d=>({ id: d.id, ...d.data() })));
  }

  async function loadProfile(uid){
    const docRef = doc(db, 'profiles', uid);
    const docSnap = await getDoc(docRef);
    if(docSnap.exists()) setProfile({ uid, ...docSnap.data() });
    else {
      // create default profile
      const defaultProfile = { name: 'New User', role: 'Student', bio: '', points: 0, earnings: 0 };
      await updateDoc(docRef, defaultProfile).catch(async ()=>{
        // if update fails (no doc), create
        await addDoc(collection(db,'profiles'), { uid, ...defaultProfile });
        setProfile({ uid, ...defaultProfile });
      });
    }
  }

  // ---------- ACTIONS ----------
  async function signUp(email, password, displayName){
    const cred = await createUserWithEmailAndPassword(auth, email, password);
    // create profile doc
    await addDoc(collection(db, 'profiles'), { uid: cred.user.uid, name: displayName || email.split('@')[0], role: 'Student', bio: '', points: 0, earnings: 0 });
    // reload profile
    await loadProfile(cred.user.uid);
  }

  async function signIn(email, password){
    await signInWithEmailAndPassword(auth, email, password);
  }

  async function signOutUser(){
    await signOut(auth);
  }

  async function submitNote(note){
    if(!user) return alert('Please sign in to contribute');
    const toSave = { ...note, authorUid: user.uid, authorName: profile?.name || 'Unknown', createdAt: Date.now() };
    await addDoc(collection(db, 'notes'), toSave);
    await loadNotes();
    // reward points
    await updateProfilePoints(user.uid, 5);
  }

  async function postJob(job){
    if(!user) return alert('Please sign in');
    const toSave = { ...job, posterUid: user.uid, createdAt: Date.now(), status: 'open' };
    await addDoc(collection(db, 'jobs'), toSave);
    await loadJobs();
  }

  async function completeJob(jobId){
    if(!user) return alert('Please sign in');
    // Mark job complete and credit earnings to current user as an example
    const jobRef = doc(db, 'jobs', jobId);
    await updateDoc(jobRef, { status: 'completed', completedBy: user.uid, completedAt: Date.now() });
    // credit user earnings (demo fixed amount or job.price)
    const jobSnapshot = await getDoc(jobRef);
    const price = jobSnapshot.exists() ? (jobSnapshot.data().price || 50) : 50;
    await updateProfileEarnings(user.uid, price);
    await loadJobs();
  }

  async function submitQuizScore(quizId, score){
    if(!user) return alert('Sign in first');
    // store attempts collection
    await addDoc(collection(db,'quizAttempts'), { quizId, uid: user.uid, score, at: Date.now() });
    await updateProfilePoints(user.uid, score);
  }

  async function updateProfilePoints(uid, delta){
    // naive approach - read then update
    const q = query(collection(db,'profiles'), where('uid','==',uid));
    const snap = await getDocs(q);
    if(snap.empty) return;
    const pdoc = snap.docs[0];
    const data = pdoc.data();
    await updateDoc(doc(db,'profiles',pdoc.id), { points: (data.points||0) + (delta||0) });
    await loadProfile(uid);
  }

  async function updateProfileEarnings(uid, amount){
    const q = query(collection(db,'profiles'), where('uid','==',uid));
    const snap = await getDocs(q);
    if(snap.empty) return;
    const pdoc = snap.docs[0];
    const data = pdoc.data();
    await updateDoc(doc(db,'profiles',pdoc.id), { earnings: (data.earnings||0) + (amount||0) });
    await loadProfile(uid);
  }

  if(loading) return <div className="p-6">Loading...</div>;

  return (
    <div className="min-h-screen bg-yellow-50 p-6">
      <div className="max-w-5xl mx-auto bg-white rounded-xl shadow">
        <Header user={profile} route={route} setRoute={setRoute} onSignIn={signIn} onSignUp={signUp} onSignOut={signOutUser} />
        <main className="p-6">
          {route === 'home' && <Home profile={profile} notes={notes.slice(0,3)} />}
          {route === 'learn' && <Learn notes={notes} onSubmit={submitNote} />}
          {route === 'work' && <Work jobs={jobs} onComplete={completeJob} onPostJob={postJob} />}
          {route === 'community' && <Community notes={notes} onSubmit={submitNote} />}
          {route === 'quiz' && <QuizList quizzes={quizzes} onSubmit={submitQuizScore} />}
          {route === 'profile' && <Profile profile={profile} />}
        </main>
      </div>
    </div>
  );
}

// ---------- UI COMPONENTS (same as before but simple) ----------
function Header({ user, route, setRoute, onSignIn, onSignUp, onSignOut }){
  const [showAuth, setShowAuth] = useState(false);
  return (
    <header className="flex items-center justify-between p-4 border-b">
      <div className="flex items-center gap-3">
        <div className="w-12 h-12 rounded-full bg-amber-400 flex items-center justify-center font-bold text-white">R</div>
        <div>
          <div className="font-bold text-lg">Rachitshikha</div>
          <div className="text-xs text-gray-600">Law learning & earning</div>
        </div>
      </div>
      <nav className="flex items-center gap-2">
        <button onClick={()=>setRoute('home')} className="px-3 py-1 rounded">Home</button>
        <button onClick={()=>setRoute('learn')} className="px-3 py-1 rounded">Learn</button>
        <button onClick={()=>setRoute('work')} className="px-3 py-1 rounded">Work</button>
        <button onClick={()=>setRoute('community')} className="px-3 py-1 rounded">Community</button>
        <button onClick={()=>setRoute('quiz')} className="px-3 py-1 rounded">Quiz</button>
        <button onClick={()=>setRoute('profile')} className="px-3 py-1 rounded">Profile</button>
        {user ? (
          <div className="ml-4 text-sm text-right">
            <div>{user.name}</div>
            <div className="text-xs text-gray-600">₹{user.earnings || 0} • {user.points || 0} pts</div>
            <div className="mt-1"><button onClick={onSignOut} className="text-red-600 text-xs">Sign out</button></div>
          </div>
        ) : (
          <AuthBox onSignIn={onSignIn} onSignUp={onSignUp} />
        )}
      </nav>
    </header>
  );
}

function AuthBox({ onSignIn, onSignUp }){
  const [show, setShow] = useState(false);
  const [email, setEmail] = useState('');
  const [pass, setPass] = useState('');
  const [name, setName] = useState('');

  return (
    <div>
      <button onClick={()=>setShow(!show)} className="px-3 py-1 rounded bg-amber-200">Sign In / Up</button>
      {show && (
        <div className="p-3 mt-2 border rounded bg-white absolute right-6">
          <input placeholder="Name (for sign up)" value={name} onChange={e=>setName(e.target.value)} className="block mb-2 p-1 border" />
          <input placeholder="Email" value={email} onChange={e=>setEmail(e.target.value)} className="block mb-2 p-1 border" />
          <input placeholder="Password" type="password" value={pass} onChange={e=>setPass(e.target.value)} className="block mb-2 p-1 border" />
          <div className="flex gap-2">
            <button onClick={()=>onSignIn(email, pass)} className="px-3 py-1 rounded bg-green-200">Sign In</button>
            <button onClick={()=>onSignUp(email, pass, name)} className="px-3 py-1 rounded bg-blue-200">Sign Up</button>
          </div>
        </div>
      )}
    </div>
  );
}

function Home({ profile, notes }){
  return (
    <section>
      <h2 className="text-2xl font-bold mb-3">Welcome {profile?.name || 'Guest'}</h2>
      <div className="grid md:grid-cols-3 gap-4">
        <div className="md:col-span-2">
          <div className="mb-4">
            <div className="font-semibold">Quick actions</div>
          </div>
          <div>
            <div className="font-semibold mb-2">Latest notes</div>
            {notes.map(n=> (
              <div key={n.id} className="p-2 border-b">
                <div className="font-medium">{n.title}</div>
                <div className="text-sm text-gray-600">{n.preview}</div>
              </div>
            ))}
          </div>
        </div>
        <aside>
          <div className="p-3 border rounded">
            <div className="font-semibold">Profile</div>
            <div className="text-sm">{profile?.role}</div>
            <div className="text-sm">Earnings: ₹{profile?.earnings || 0}</div>
            <div className="text-sm">Points: {profile?.points || 0}</div>
          </div>
        </aside>
      </div>
    </section>
  );
}

function Learn({ notes, onSubmit }){
  const [open, setOpen] = useState(false);
  const [form, setForm] = useState({ title: '', preview: '', content: '' });
  function submit(){
    if(!form.title) return alert('Enter title');
    onSubmit({ ...form });
    setForm({ title: '', preview: '', content: '' });
    setOpen(false);
  }
  return (
    <section>
      <div className="flex justify-between items-center mb-4">
        <h2 className="text-xl font-bold">Notes & Articles</h2>
        <button onClick={()=>setOpen(!open)} className="px-3 py-1 rounded bg-amber-200">Contribute</button>
      </div>
      {open && (
        <div className="mb-4 p-3 border rounded">
          <input value={form.title} onChange={e=>setForm({...form,title:e.target.value})} placeholder="Title" className="w-full mb-2 p-2 border" />
          <input value={form.preview} onChange={e=>setForm({...form,preview:e.target.value})} placeholder="Preview" className="w-full mb-2 p-2 border" />
          <textarea value={form.content} onChange={e=>setForm({...form,content:e.target.value})} placeholder="Content" className="w-full mb-2 p-2 border h-32" />
          <div className="text-right"><button onClick={submit} className="px-3 py-1 rounded bg-green-200">Submit</button></div>
        </div>
      )}

      <div>
        {notes.map(n=> (
          <div key={n.id} className="p-3 border-b">
            <div className="font-semibold">{n.title}</div>
            <div className="text-sm text-gray-600">{n.preview}</div>
          </div>
        ))}
      </div>
    </section>
  );
}

function Work({ jobs, onComplete, onPostJob }){
  const [openPost, setOpenPost] = useState(false);
  const [jobForm, setJobForm] = useState({ title: '', desc: '', price: 50 });
  async function post(){
    if(!jobForm.title) return alert('Title required');
    await onPostJob(jobForm);
    setJobForm({ title: '', desc: '', price: 50 });
    setOpenPost(false);
  }
  return (
    <section>
      <div className="flex justify-between items-center mb-4">
        <h2 className="text-xl font-bold">Work & Earn</h2>
        <button onClick={()=>setOpenPost(!openPost)} className="px-3 py-1 rounded bg-amber-200">Post Gig</button>
      </div>
      {openPost && (
        <div className="p-3 border rounded mb-4">
          <input value={jobForm.title} onChange={e=>setJobForm({...jobForm,title:e.target.value})} placeholder="Title" className="w-full mb-2 p-2 border" />
          <input value={jobForm.desc} onChange={e=>setJobForm({...jobForm,desc:e.target.value})} placeholder="Desc" className="w-full mb-2 p-2 border" />
          <input type="number" value={jobForm.price} onChange={e=>setJobForm({...jobForm,price:parseInt(e.target.value||0)})} className="w-full mb-2 p-2 border" />
          <div className="text-right"><button onClick={post} className="px-3 py-1 rounded bg-green-200">Post</button></div>
        </div>
      )}

      <div>
        {jobs.map(job=> (
          <div key={job.id} className="p-3 border-b">
            <div className="flex justify-between items-center">
              <div>
                <div className="font-semibold">{job.title} — ₹{job.price}</div>
                <div className="text-sm text-gray-600">{job.desc}</div>
              </div>
              <div>
                {job.status === 'open' ? <button onClick={()=>onComplete(job.id)} className="px-3 py-1 rounded bg-blue-200">Complete & Claim</button> : <div className="text-sm text-green-600">Completed</div>}
              </div>
            </div>
          </div>
        ))}
      </div>
    </section>
  );
}

function Community({ notes, onSubmit }){
  return (
    <section>
      <h2 className="text-xl font-bold mb-3">Community</h2>
      <div>
        {notes.map(n=> (
          <div key={n.id} className="p-3 border-b">
            <div className="font-semibold">{n.title}</div>
            <div className="text-sm text-gray-600">{n.preview}</div>
          </div>
        ))}
      </div>
    </section>
  );
}

function QuizList({ quizzes, onSubmit }){
  return (
    <section>
      <h2 className="text-xl font-bold mb-3">Quizzes</h2>
      <div>
        {quizzes.map(q=> (
          <div key={q.id} className="p-3 border-b">
            <div className="font-semibold">{q.title}</div>
            <div className="text-sm text-gray-600">{q.desc}</div>
            <div className="mt-2"><Quiz quiz={q} onSubmit={(score)=>onSubmit(q.id, score)} /></div>
          </div>
        ))}
      </div>
    </section>
  );
}

function Quiz({ quiz, onSubmit }){
  const [answers, setAnswers] = useState({});
  const [done, setDone] = useState(false);
  function answer(qid, opt){ setAnswers(prev=>({...prev,[qid]:opt})); }
  function submit(){
    let score = 0; quiz.questions.forEach(q=>{ if(answers[q.id]===q.correct) score += q.points||1 });
    setDone(true);
    onSubmit(score);
  }
  return (
    <div>
      {quiz.questions.map(q=> (
        <div key={q.id} className="mb-2">
          <div className="font-medium">{q.text}</div>
          <div className="text-sm">
            {q.options.map(o=> (
              <label key={o} className="mr-3"><input type="radio" name={q.id} checked={answers[q.id]===o} onChange={()=>answer(q.id,o)} /> {o}</label>
            ))}
          </div>
        </div>
      ))}
      {!done ? <button onClick={submit} className="px-3 py-1 rounded bg-green-200">Submit</button> : <div className="text-sm text-gray-600">Score submitted</div>}
    </div>
  );
}

function Profile({ profile }){
  return (
    <section>
      <h2 className="text-xl font-bold mb-3">Profile</h2>
      <div className="p-3 border rounded">
        <div className="font-semibold">{profile?.name}</div>
        <div className="text-sm">{profile?.role}</div>
        <div className="text-sm">Earnings: ₹{profile?.earnings || 0}</div>
        <div className="text-sm">Points: {profile?.points || 0}</div>
      </div>
    </section>
  );
}

/*
-- Firestore Collections used --
- notes: { title, preview, content, authorUid, authorName, createdAt }
- jobs: { title, desc, price, posterUid, status, createdAt }
- quizzes: { title, desc, questions: [...] }
- profiles: { uid, name, role, bio, points, earnings }
- quizAttempts: { quizId, uid, score, at }

Security (development):
- For quick testing set Firestore rules to allow reads/writes. BEFORE production, add rules restricting updates to profile owners and authenticated users only.

Suggested Firestore rules (starter):
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read: if true;
      allow write: if request.auth != null; // dev only
    }
  }
}

Remember to tighten these rules for production.
*/

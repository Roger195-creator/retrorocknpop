# retrorocknpop
Concours de talents de la chanson rétro
import React, { useState, useEffect } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, collection, onSnapshot, addDoc, serverTimestamp, query, orderBy } from 'firebase/firestore';

// Configuration de Firebase. Ces variables sont automatiquement fournies par l'environnement Canvas.
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

// Composant principal de l'application
const App = () => {
  // États pour gérer l'application
  const [participants, setParticipants] = useState([]);
  const [nom, setNom] = useState('');
  const [email, setEmail] = useState('');
  const [db, setDb] = useState(null);
  const [auth, setAuth] = useState(null);
  const [userId, setUserId] = useState(null);
  const [message, setMessage] = useState('');

  // Initialisation de Firebase et authentification
  useEffect(() => {
    if (Object.keys(firebaseConfig).length === 0) {
        console.error("Firebase config is missing.");
        return;
    }

    const app = initializeApp(firebaseConfig);
    const firestore = getFirestore(app);
    const authentication = getAuth(app);
    setDb(firestore);
    setAuth(authentication);

    // Établir l'authentification
    const setupAuth = async () => {
      try {
        if (initialAuthToken) {
          await signInWithCustomToken(authentication, initialAuthToken);
        } else {
          // Si le jeton n'est pas disponible, se connecter de manière anonyme
          await signInAnonymously(authentication);
        }
      } catch (error) {
        console.error("Erreur d'authentification:", error);
      }
    };
    setupAuth();

    // Gérer les changements d'état d'authentification et obtenir l'ID de l'utilisateur
    const unsubscribeAuth = onAuthStateChanged(authentication, (user) => {
      if (user) {
        setUserId(user.uid);
      } else {
        setUserId(null);
      }
    });

    return () => {
      unsubscribeAuth();
    };
  }, []);

  // Écouter les mises à jour en temps réel des participants depuis Firestore
  useEffect(() => {
    if (!db) return;

    // Le chemin de la collection doit inclure l'ID de l'application et "public"
    // pour permettre le partage des données entre tous les utilisateurs.
    const collectionPath = `/artifacts/${appId}/public/data/participants`;

    // Créer une requête pour la collection.
    // NOTE: Il est préférable de trier en mémoire pour éviter les erreurs d'index.
    const q = query(collection(db, collectionPath));

    const unsubscribe = onSnapshot(q, (querySnapshot) => {
      let participantsList = [];
      querySnapshot.forEach((doc) => {
        participantsList.push({ id: doc.id, ...doc.data() });
      });
      // Trier la liste par la date d'inscription
      const sortedList = participantsList.sort((a, b) => {
        if (!a.date || !b.date) return 0;
        return b.date.toDate().getTime() - a.date.toDate().getTime();
      });
      setParticipants(sortedList);
    }, (error) => {
      console.error("Erreur lors de la récupération des données:", error);
      setMessage("Erreur lors du chargement des participants.");
    });

    // Nettoyer l'abonnement lorsque le composant est démonté
    return () => unsubscribe();
  }, [db, userId, appId]);

  // Gérer la soumission du formulaire
  const handleSubmit = async (e) => {
    e.preventDefault();
    if (!db || !nom.trim() || !email.trim()) {
      setMessage("Veuillez remplir tous les champs.");
      return;
    }

    try {
      // Le chemin de la collection doit être le même que celui utilisé pour la lecture
      const collectionPath = `/artifacts/${appId}/public/data/participants`;
      await addDoc(collection(db, collectionPath), {
        nom: nom,
        email: email,
        date: serverTimestamp(),
        userId: userId
      });
      setMessage("Inscription réussie !");
      setNom('');
      setEmail('');
    } catch (error) {
      console.error("Erreur lors de l'ajout du document:", error);
      setMessage("Erreur lors de l'inscription. Veuillez réessayer.");
    }
  };

  // Styles pour la mise en page
  const mainStyle = "bg-gradient-to-br from-blue-100 to-purple-100 min-h-screen p-4 sm:p-8 flex items-center justify-center font-sans text-gray-800";
  const containerStyle = "max-w-4xl w-full bg-white rounded-3xl shadow-xl overflow-hidden flex flex-col md:flex-row";
  const formSectionStyle = "w-full md:w-1/2 p-6 sm:p-10 flex flex-col justify-center";
  const listSectionStyle = "w-full md:w-1/2 p-6 sm:p-10 bg-gray-50 overflow-y-auto max-h-[500px] md:max-h-full";
  const titleStyle = "text-3xl sm:text-4xl font-extrabold text-center text-blue-800 mb-6";
  const subTitleStyle = "text-xl sm:text-2xl font-bold text-blue-700 mb-4";
  const labelStyle = "block text-gray-700 font-semibold mb-2";
  const inputStyle = "w-full p-3 mb-4 border border-gray-300 rounded-xl focus:outline-none focus:ring-2 focus:ring-blue-500 transition-all";
  const buttonStyle = "w-full bg-blue-600 text-white font-bold p-3 rounded-xl hover:bg-blue-700 transition-colors shadow-lg";
  const messageStyle = "mt-4 text-center font-medium";
  const participantItemStyle = "bg-white p-4 rounded-xl shadow-md mb-3 transition-transform transform hover:scale-105";

  return (
    <div className={mainStyle}>
      <div className={containerStyle}>
        
        {/* Section du formulaire d'inscription */}
        <div className={formSectionStyle}>
          <h2 className={titleStyle}>Participez au Concours !</h2>
          <form onSubmit={handleSubmit}>
            <div className="mb-4">
              <label htmlFor="nom" className={labelStyle}>Nom</label>
              <input
                id="nom"
                type="text"
                value={nom}
                onChange={(e) => setNom(e.target.value)}
                className={inputStyle}
                placeholder="Votre nom"
              />
            </div>
            <div className="mb-4">
              <label htmlFor="email" className={labelStyle}>Email</label>
              <input
                id="email"
                type="email"
                value={email}
                onChange={(e) => setEmail(e.target.value)}
                className={inputStyle}
                placeholder="Votre email"
              />
            </div>
            <button type="submit" className={buttonStyle}>
              S'inscrire
            </button>
          </form>
          {message && <div className={`${messageStyle} ${message.includes('réussie') ? 'text-green-600' : 'text-red-600'}`}>{message}</div>}
        </div>

        {/* Section de la liste des participants */}
        <div className={listSectionStyle}>
          <h2 className={subTitleStyle}>Liste des Participants ({participants.length})</h2>
          <div className="space-y-3">
            {participants.map((p) => (
              <div key={p.id} className={participantItemStyle}>
                <p className="font-bold text-gray-900">{p.nom}</p>
                <p className="text-sm text-gray-600 truncate">{p.email}</p>
              </div>
            ))}
          </div>
        </div>

      </div>
    </div>
  );
};

export default App;

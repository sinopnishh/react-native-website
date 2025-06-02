import React, { useState, useEffect, useRef } from 'react';

// Firebase modüllerini import edin
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, setDoc, collection, query, onSnapshot, orderBy, limit } from 'firebase/firestore';

// Global değişkenleri kontrol edin ve kullanın
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

// Firebase uygulamasını başlatın
const app = initializeApp(firebaseConfig);
const db = getFirestore(app);
const auth = getAuth(app);

// Yapay zeka backend'inizin API adresi.
// BURAYI KENDİ BACKEND SUNUCUNUZUN ADRESİYLE DEĞİŞTİRMELİSİNİZ!
// Örnek: const AI_API_URL = 'https://your-ai-backend.com/process-image';
const AI_API_URL = 'https://your-ai-backend-url.com/process-image'; // Bu URL'yi kendi backend'inizle değiştirin

// Özel Modal Bileşeni
const CustomModal = ({ visible, message, onClose }) => {
  if (!visible) return null;
  return (
    <div className="fixed inset-0 bg-gray-600 bg-opacity-75 flex items-center justify-center z-50">
      <div className="bg-white p-6 rounded-lg shadow-xl max-w-sm w-full text-center">
        <p className="text-lg font-semibold mb-4 text-gray-800">{message}</p>
        <button
          onClick={onClose}
          className="bg-blue-500 hover:bg-blue-600 text-white font-bold py-2 px-4 rounded-full transition duration-300 ease-in-out"
        >
          Tamam
        </button>
      </div>
    </div>
  );
};

export default function App() {
  const [selectedImage, setSelectedImage] = useState(null); // Seçilen görselin Data URL'si (Base64)
  const [hairDescription, setHairDescription] = useState(''); // Kullanıcının saç açıklaması
  const [loading, setLoading] = useState(false); // Yükleme durumu
  const [generatedVariations, setGeneratedVariations] = useState([]); // Oluşturulan varyasyonlar
  const [userId, setUserId] = useState(null); // Firestore için kullanıcı ID'si
  const [isAuthReady, setIsAuthReady] = useState(false); // Firebase Auth hazır mı?
  const [history, setHistory] = useState([]); // İşlem geçmişi

  const fileInputRef = useRef(null); // Dosya input elementine referans

  // Modal durumu
  const [modalVisible, setModalVisible] = useState(false);
  const [modalMessage, setModalMessage] = useState('');

  const showAlert = (title, message) => {
    setModalMessage(`${title}\n${message}`);
    setModalVisible(true);
  };

  // Firebase kimlik doğrulama ve Firestore kurulumu
  useEffect(() => {
    const setupFirebase = async () => {
      try {
        if (initialAuthToken) {
          await signInWithCustomToken(auth, initialAuthToken);
          console.log("Signed in with custom token.");
        } else {
          await signInAnonymously(auth);
          console.log("Signed in anonymously.");
        }
      } catch (error) {
        console.error("Firebase Auth Error:", error);
        showAlert("Hata", "Firebase kimlik doğrulama hatası: " + error.message);
      }
    };

    const unsubscribe = onAuthStateChanged(auth, (user) => {
      if (user) {
        setUserId(user.uid);
        console.log("User ID:", user.uid);
      } else {
        // Eğer anonim giriş başarısız olursa veya kullanıcı yoksa, rastgele bir ID kullan
        setUserId(crypto.randomUUID());
        console.log("Anonymous user ID:", userId);
      }
      setIsAuthReady(true); // Kimlik doğrulama hazır
    });

    setupFirebase();
    return () => unsubscribe();
  }, []);

  // Firestore'dan geçmişi dinleme
  useEffect(() => {
    if (!isAuthReady || !userId) return;

    // Firestore güvenlik kurallarına uygun olarak public veya private koleksiyonu seçin
    // Bu örnekte, kullanıcının kendi geçmişini tutmak için private koleksiyon kullanıyoruz.
    const historyCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/image_modifications`);
    const q = query(historyCollectionRef, orderBy("timestamp", "desc"), limit(5)); // Son 5 işlemi göster

    const unsubscribe = onSnapshot(q, (snapshot) => {
      const fetchedHistory = snapshot.docs.map(doc => ({
        id: doc.id,
        ...doc.data()
      }));
      setHistory(fetchedHistory);
      console.log("History updated:", fetchedHistory);
    }, (error) => {
      console.error("Firestore history snapshot error:", error);
      showAlert("Hata", "Geçmiş yüklenirken bir sorun oluştu: " + error.message);
    });

    return () => unsubscribe();
  }, [isAuthReady, userId]);

  // Görsel seçme fonksiyonu (Web için input type="file" kullanır)
  const handleImageSelect = (event) => {
    const file = event.target.files[0];
    if (file) {
      const reader = new FileReader();
      reader.onloadend = () => {
        setSelectedImage(reader.result); // Base64 Data URL
        setGeneratedVariations([]); // Yeni görsel seçildiğinde varyasyonları sıfırla
      };
      reader.readAsDataURL(file);
    }
  };

  // Görseli işleme ve AI backend'ine gönderme fonksiyonu
  const processImage = async () => {
    if (!selectedImage) {
      showAlert("Hata", "Lütfen önce bir görsel seçin.");
      return;
    }
    if (!hairDescription.trim()) {
      showAlert("Hata", "Lütfen saç için bir açıklama girin (örn: 'uzun ve sarı').");
      return;
    }
    if (!AI_API_URL || AI_API_URL === 'https://your-ai-backend-url.com/process-image') {
      showAlert("Hata", "Lütfen AI_API_URL'yi kendi backend adresinizle güncelleyin.");
      return;
    }

    setLoading(true);
    setGeneratedVariations([]); // Yeni işlemde eski varyasyonları temizle

    try {
      // Base64 Data URL'den sadece Base64 kısmını al
      const base64Image = selectedImage.split(',')[1];

      // Backend'e gönderilecek veri
      const payload = {
        image: base64Image,
        description: hairDescription,
        // userId: userId, // Backend'de kullanıcı takibi için gönderilebilir
      };

      console.log("Sending payload to AI backend...");
      // Yapay zeka backend'ine POST isteği gönder
      const response = await fetch(AI_API_URL, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify(payload),
      });

      if (!response.ok) {
        const errorData = await response.json();
        throw new Error(`API Hatası: ${response.status} - ${errorData.message || response.statusText}`);
      }

      const result = await response.json();
      console.log("AI response received:", result);

      if (result.variations && Array.isArray(result.variations) && result.variations.length > 0) {
        setGeneratedVariations(result.variations);

        // İşlem geçmişini Firestore'a kaydet
        if (userId) {
          const historyCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/image_modifications`);
          await setDoc(doc(historyCollectionRef), { // doc() ile otomatik ID oluşturulur
            originalImage: selectedImage, // Data URL olarak kaydediyoruz
            description: hairDescription,
            generatedVariations: result.variations,
            timestamp: new Date().toISOString(),
            userId: userId, // Kullanıcı ID'sini de kaydedelim
          });
          console.log("Operation saved to Firestore.");
        } else {
          console.warn("User ID not available, skipping Firestore save.");
        }
      } else {
        showAlert("Sonuç Yok", "Yapay zeka beklenen varyasyonları döndürmedi.");
      }
    } catch (error) {
      console.error("Görsel işleme hatası:", error);
      showAlert("Hata", "Görsel işlenirken bir sorun oluştu: " + error.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="min-h-screen bg-gray-100 flex flex-col items-center py-8 font-['Inter']">
      {/* Tailwind CSS'i yükle */}
      <script src="https://cdn.tailwindcss.com"></script>

      <CustomModal visible={modalVisible} message={modalMessage} onClose={() => setModalVisible(false)} />

      <h1 className="text-4xl font-bold text-gray-800 mb-6 mt-4">Saç Değişim Uygulaması</h1>
      {userId && isAuthReady && (
        <p className="text-sm text-gray-600 mb-4">Kullanıcı ID: {userId}</p>
      )}

      {/* Gizli dosya inputu */}
      <input
        type="file"
        accept="image/*"
        ref={fileInputRef}
        onChange={handleImageSelect}
        className="hidden"
      />

      <button
        onClick={() => fileInputRef.current.click()} // Butona tıklandığında dosya inputunu tetikle
        className="bg-blue-500 hover:bg-blue-600 text-white font-bold py-3 px-8 rounded-full shadow-lg transition duration-300 ease-in-out mb-6"
      >
        Görsel Seç
      </button>

      {selectedImage && (
        <div className="w-4/5 md:w-3/5 lg:w-2/5 mb-6 bg-gray-200 rounded-xl overflow-hidden border border-gray-300 shadow-md">
          <img src={selectedImage} alt="Seçilen Görsel" className="w-full h-auto object-contain rounded-xl" />
        </div>
      )}

      <input
        type="text"
        placeholder="Saç nasıl olsun? (örn: 'uzun ve sarı')"
        placeholderTextColor="#999"
        value={hairDescription}
        onChange={(e) => setHairDescription(e.target.value)}
        className="w-4/5 md:w-3/5 lg:w-2/5 p-4 border border-gray-300 rounded-xl mb-6 text-lg text-gray-700 focus:outline-none focus:ring-2 focus:ring-blue-500"
      />

      <button
        onClick={processImage}
        disabled={loading}
        className={`bg-green-500 hover:bg-green-600 text-white font-bold py-3 px-8 rounded-full shadow-lg transition duration-300 ease-in-out ${loading ? 'opacity-60 cursor-not-allowed' : ''} mb-10`}
      >
        {loading ? (
          <div className="flex items-center justify-center">
            <svg className="animate-spin -ml-1 mr-3 h-5 w-5 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
              <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle>
              <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
            </svg>
            İşleniyor...
          </div>
        ) : (
          'Saçları Değiştir'
        )}
      </button>

      {generatedVariations.length > 0 && (
        <div className="w-full max-w-4xl p-6 bg-white rounded-xl shadow-md mb-10">
          <h2 className="text-3xl font-bold text-gray-800 mb-6 text-center">Oluşturulan Varyasyonlar:</h2>
          <div className="grid grid-cols-1 md:grid-cols-2 gap-6 justify-items-center">
            {generatedVariations.map((base64Image, index) => (
              <div key={index} className="w-full bg-gray-200 rounded-xl overflow-hidden border border-gray-300 shadow-md">
                <img
                  src={`data:image/png;base64,${base64Image}`}
                  alt={`Varyasyon ${index + 1}`}
                  className="w-full h-auto object-contain rounded-xl"
                />
              </div>
            ))}
          </div>
        </div>
      )}

      {history.length > 0 && (
        <div className="w-full max-w-4xl p-6 bg-white rounded-xl shadow-md">
          <h2 className="text-3xl font-bold text-gray-800 mb-6 text-center">Son İşlemler:</h2>
          <div className="space-y-4">
            {history.map((item) => (
              <div key={item.id} className="bg-gray-50 p-4 rounded-lg shadow-sm border border-gray-200">
                <p className="text-lg font-semibold text-gray-700 mb-2">Açıklama: {item.description}</p>
                <div className="flex flex-wrap justify-center gap-4 mb-3">
                  {item.originalImage && (
                    <img src={item.originalImage} alt="Orijinal" className="w-24 h-20 object-cover rounded-md border border-gray-300" />
                  )}
                  {item.generatedVariations && item.generatedVariations.length > 0 && (
                    item.generatedVariations.map((base64Image, varIndex) => (
                      <img
                        key={varIndex}
                        src={`data:image/png;base64,${base64Image}`}
                        alt={`Varyasyon ${varIndex + 1}`}
                        className="w-24 h-20 object-cover rounded-md border border-gray-300"
                      />
                    ))
                  )}
                </div>
                <p className="text-sm text-gray-500 text-right">{new Date(item.timestamp).toLocaleString()}</p>
              </div>
            

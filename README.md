// Collections structure
users: {
  userId: {
    nama: "Ahmad bin Ali",
    noKP: "851212145678",
    email: "ahmad@example.com",
    telefon: "0123456789",
    alamat: "No 1, Jalan...",
    peranan: "hakim", // hakim, pendaftar, pengerusi_jkp, pegawai_agama, plaintif, defendan, wakil_hakam_plaintif, wakil_hakam_defendan, hakam_panel_plaintif, hakam_panel_defendan
    status: "active",
    createdAt: timestamp
  }
}

cases: {
  caseId: {
    noKes: "HAKAM/2024/001",
    plaintif: userId,
    defendan: userId,
    status: "jkp_proses", // jkp_proses, jkp_selesai, hakam_proses, hakam_selesai
    createdAt: timestamp
  }
}

jkp_records: {
  recordId: {
    caseId: "case123",
    wakil_plaintif: { nama, noKP, alamat, telefon, email, kadPengenalan_url },
    wakil_defendan: { nama, noKP, alamat, telefon, email },
    keputusan: "berjaya", // berjaya, gagal
    laporan: "text laporan...",
    createdAt: timestamp
  }
}# Create new project
npx create-expo-app HakamPro --template blank-typescript
cd HakamPro

# Install dependencies
npm install @react-navigation/native @react-navigation/stack
npm install react-native-paper
npm install firebase
npm install @react-native-google-signin/google-signin
npm install react-native-calendar-events

# Expo specific
npx expo install @react-native-async-storage/async-storage
npx expo install expo-document-picker
npx expo install expo-file-system
src/
├── components/
│   ├── auth/
│   │   ├── LoginScreen.tsx
│   │   └── RegisterScreen.tsx
│   ├── modules/
│   │   ├── jkp/
│   │   │   ├── JKPForm.tsx
│   │   │   ├── JKPSurat.tsx
│   │   │   └── JKPLaporan.tsx
│   │   ├── hakam-keluarga/
│   │   │   ├── HakamKeluargaForm.tsx
│   │   │   └── HakamLaporan.tsx
│   │   ├── hakam-panel/
│   │   │   ├── PanelDashboard.tsx
│   │   │   ├── PanelCalendar.tsx
│   │   │   └── PanelLaporan.tsx
│   │   └── statistik/
│   │       ├── StatistikJKP.tsx
│   │       └── StatistikHakam.tsx
│   └── common/
│       ├── Header.tsx
│       ├── Sidebar.tsx
│       └── PDFGenerator.tsx
├── services/
│   ├── firebase.ts
│   ├── auth.ts
│   ├── database.ts
│   └── pdfService.ts
└── utils/
    ├── constants.ts
    └── helpers.ts
    import { createUserWithEmailAndPassword, signInWithEmailAndPassword } from 'firebase/auth';
import { doc, setDoc } from 'firebase/firestore';
import { auth, db } from './firebase';

export const registerUser = async (userData: {
  nama: string;
  noKP: string;
  email: string;
  telefon: string;
  alamat: string;
  peranan: string;
  password: string;
}) => {
  try {
    // Create auth user
    const userCredential = await createUserWithEmailAndPassword(auth, userData.email, userData.password);
    
    // Create user document
    await setDoc(doc(db, 'users', userCredential.user.uid), {
      ...userData,
      createdAt: new Date(),
      status: 'active'
    });
    
    return userCredential.user;
  } catch (error) {
    throw error;
  }
};
import React, { useState } from 'react';
import { View, StyleSheet } from 'react-native';
import { TextInput, Button, Card, Title } from 'react-native-paper';
import { storage, db } from '../../../services/firebase';
import { ref, uploadBytes, getDownloadURL } from 'firebase/storage';
import { doc, setDoc } from 'firebase/firestore';
import * as DocumentPicker from 'expo-document-picker';

const JKPForm = ({ caseId, userRole }) => {
  const [formData, setFormData] = useState({
    nama: '',
    noKP: '',
    alamat: '',
    telefon: '',
    email: ''
  });
  const [kadPengenalan, setKadPengenalan] = useState(null);

  const pickDocument = async () => {
    try {
      const result = await DocumentPicker.getDocumentAsync({
        type: 'image/*',
        copyToCacheDirectory: true
      });
      
      if (!result.canceled) {
        setKadPengenalan(result.assets[0]);
      }
    } catch (error) {
      console.error('Error picking document:', error);
    }
  };

  const uploadDocument = async () => {
    if (!kadPengenalan) return null;

    const response = await fetch(kadPengenalan.uri);
    const blob = await response.blob();
    const storageRef = ref(storage, `kad_pengenalan/${Date.now()}_${formData.noKP}`);
    
    await uploadBytes(storageRef, blob);
    return await getDownloadURL(storageRef);
  };

  const submitForm = async () => {
    try {
      const kadPengenalanUrl = await uploadDocument();
      
      const jkpData = {
        ...formData,
        kadPengenalan_url: kadPengenalanUrl,
        caseId: caseId,
        createdAt: new Date(),
        submittedBy: userRole
      };

      await setDoc(doc(db, 'jkp_records', `${caseId}_${Date.now()}`), jkpData);
      alert('Borang JKP berjaya dihantar!');
    } catch (error) {
      alert('Error: ' + error.message);
    }
  };

  return (
    <Card style={styles.card}>
      <Card.Content>
        <Title>Borang Wakil Jawatankuasa Pendamai</Title>
        
        <TextInput
          label="Nama Penuh"
          value={formData.nama}
          onChangeText={text => setFormData({...formData, nama: text})}
          style={styles.input}
        />
        
        <TextInput
          label="No. Kad Pengenalan"
          value={formData.noKP}
          onChangeText={text => setFormData({...formData, noKP: text})}
          style={styles.input}
        />
        
        <TextInput
          label="Alamat"
          value={formData.alamat}
          onChangeText={text => setFormData({...formData, alamat: text})}
          multiline
          style={styles.input}
        />
        
        <TextInput
          label="No. Telefon"
          value={formData.telefon}
          onChangeText={text => setFormData({...formData, telefon: text})}
          style={styles.input}
        />
        
        <TextInput
          label="Emel"
          value={formData.email}
          onChangeText={text => setFormData({...formData, email: text})}
          style={styles.input}
        />
        
        <Button mode="outlined" onPress={pickDocument} style={styles.button}>
          Muat Naik Kad Pengenalan
        </Button>
        
        <Button mode="contained" onPress={submitForm} style={styles.button}>
          Hantar Borang
        </Button>
      </Card.Content>
    </Card>
  );
};
// services/pdfService.ts
export const generateSuratMahkamah = async (templateType: string, data: any) => {
  // Implementation for PDF generation using react-native-print
  // or external service like PDF.co
};
// firebase.json
{
  "firestore": {
    "rules": "firestore.rules",
    "indexes": "firestore.indexes.json"
  },
  "storage": {
    "rules": "storage.rules"
  }
}// firestore.rules
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
      allow read: if request.auth != null && get(/databases/$(database)/documents/users/$(request.auth.uid)).data.peranan in ['hakim', 'pendaftar'];
    }
    
    match /cases/{caseId} {
      allow read, write: if request.auth != null;
    }
  }
}

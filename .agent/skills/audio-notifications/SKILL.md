---
name: audio-notifications
description: Padrões de design e implementação para notificações sonoras no CreAuto. Use sempre que precisar implementar alertas sonoros no frontend baseados em eventos do sistema (sucesso, erro, interrupção). Garante consistência estética e técnica usando Web Audio API.
---

# Audio Notifications (CreAuto Standard)

Este skill define os padrões para notificações sonoras na plataforma CreAuto, utilizando a **Web Audio API** para síntese sonora resiliente (sem dependência de arquivos externos) e gerenciamento de estado de áudio.

## 🎧 Identidade Sonora (Sons Padrão)

O CreAuto utiliza três assinaturas sonoras distintas sintetizadas em tempo real:

### 1. Notificação A: Conclusão de Pesquisa (e2doc)
- **Caráter**: Positivo, discreto, informativo.
- **Implementação**: Onda senoidal (Sine) curta com rampa de ganho suave.
- **Frequência**: 880Hz (A5) -> 1320Hz (E6).
- **Duração**: 150ms.

### 2. Notificação B: Pausa por Erro / HITL
- **Caráter**: Alerta, urgente (mas não agressivo), requer atenção.
- **Implementação**: Onda quadrada (Square) com filtro passa-baixa, dois tons descendentes.
- **Frequência**: 440Hz (A4) seguido de 330Hz (E4).
- **Duração**: 400ms (200ms cada tom).

### ### 3. Notificação C: Conclusão de Cadastramento (SIC)
- **Caráter**: Recompensa, finalização, sucesso maior.
- **Implementação**: Arpejo ascendente em onda senoidal ou triângular.
- **Frequência**: 523Hz (C5) -> 659Hz (E5) -> 783Hz (G5).
- **Duração**: 450ms (150ms cada nota).

---

## 🛠️ Implementação Recomendada (React Hook)

Sempre utilize o padrão de hook abaixo para garantir que o áudio respeite o estado global de **Volume** e **Mudo**.

### Código de Referência: `useAudioNotification.ts`

```typescript
import { useState, useEffect, useCallback } from 'react';

type SoundType = 'SUCCESS_E2DOC' | 'ERROR_HITL' | 'SUCCESS_SIC';

export const useAudioNotification = () => {
  const [audioCtx, setAudioCtx] = useState<AudioContext | null>(null);
  const [volume, setVolume] = useState(0.5); // 0.0 a 1.0
  const [isMuted, setIsMuted] = useState(false);

  useEffect(() => {
    // Inicialização lazy para evitar bloqueio de autoplayer
    const initCtx = () => {
      if (!audioCtx) setAudioCtx(new (window.AudioContext || (window as any).webkitAudioContext)());
    };
    window.addEventListener('click', initCtx, { once: true });
    return () => window.removeEventListener('click', initCtx);
  }, [audioCtx]);

  const playSound = useCallback((type: SoundType) => {
    if (!audioCtx || isMuted) return;

    if (audioCtx.state === 'suspended') {
      audioCtx.resume();
    }

    const masterGain = audioCtx.createGain();
    masterGain.gain.setValueAtTime(volume, audioCtx.currentTime);
    masterGain.connect(audioCtx.destination);

    const osc = audioCtx.createOscillator();
    const gain = audioCtx.createGain();
    osc.connect(gain);
    gain.connect(masterGain);

    const now = audioCtx.currentTime;

    switch (type) {
      case 'SUCCESS_E2DOC':
        osc.type = 'sine';
        osc.frequency.setValueAtTime(880, now);
        osc.frequency.exponentialRampToValueAtTime(1320, now + 0.15);
        gain.gain.setValueAtTime(0, now);
        gain.gain.linearRampToValueAtTime(0.2, now + 0.02);
        gain.gain.linearRampToValueAtTime(0, now + 0.15);
        osc.start(now);
        osc.stop(now + 0.15);
        break;

      case 'ERROR_HITL':
        osc.type = 'square';
        // Tom 1
        osc.frequency.setValueAtTime(440, now);
        gain.gain.setValueAtTime(0.1, now);
        // Tom 2
        osc.frequency.setValueAtTime(330, now + 0.2);
        gain.gain.linearRampToValueAtTime(0, now + 0.4);
        osc.start(now);
        osc.stop(now + 0.4);
        break;

      case 'SUCCESS_SIC':
        osc.type = 'triangle';
        [523.25, 659.25, 783.99].forEach((freq, i) => {
          osc.frequency.setValueAtTime(freq, now + i * 0.15);
        });
        gain.gain.setValueAtTime(0, now);
        gain.gain.linearRampToValueAtTime(0.2, now + 0.05);
        gain.gain.linearRampToValueAtTime(0, now + 0.45);
        osc.start(now);
        osc.stop(now + 0.45);
        break;
    }
  }, [audioCtx, volume, isMuted]);

  return { playSound, setVolume, setIsMuted, isMuted, volume };
};
```

## 📋 Regras de Ouro (MANDATORY)

1. **Ativação por Usuário**: O `AudioContext` do navegador começa bloqueado. O padrão deve sempre aguardar uma primeira interação (clique) do usuário no dashboard para inicializar o áudio.
2. **Não use Arquivos Externos**: Para notificações de sistema, use síntese (`OscillatorNode`). Isso evita latência de rede e falhas de carregamento de assets.
3. **Feedback Visual**: Sempre acompanhe o som de uma notificação visual (Toast ou alteração de cor no Card) para garantir acessibilidade a usuários surdos.
4. **Volume Persistente**: Salve as preferências de volume e mudo no Supabase (perfil do usuário) ou LocalStorage.

## 🚀 Como Acionar

No frontend, integre o hook ao listener de eventos (ex: Supabase Realtime):

```typescript
const { playSound } = useAudioNotification();

useEffect(() => {
  const channel = supabase
    .channel('automation_logs')
    .on('postgres_changes', { event: 'INSERT', schema: 'public', table: 'logs' }, payload => {
       if (payload.new.type === 'E2DOC_COMPLETE') playSound('SUCCESS_E2DOC');
       if (payload.new.type === 'HITL_REQUIRED') playSound('ERROR_HITL');
    })
    .subscribe();
}, []);
```

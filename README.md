# Quebra-cabe-a-da-minha-pretinhaaa
import React, { useState, useEffect, useCallback, useMemo } from 'react';
import { Button } from '@/components/ui/button';
import { Slider } from '@/components/ui/slider';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Heart, Puzzle as PuzzleIcon, Shuffle, Check, Clock, BrainCircuit, Image as ImageIcon, Upload } from 'lucide-react';
import { Dialog, DialogContent, DialogHeader, DialogTitle, DialogTrigger } from '@/components/ui/dialog';
import { motion, AnimatePresence } from 'framer-motion';
import Confetti from 'react-confetti';
import { useWindowSize } from 'react-use';

// Paths
const DEFAULT_LOCAL_IMAGE = '/lg.jpg';
const FALLBACK_IMAGE = 'https://via.placeholder.com/1000x1000.png?text=Imagem+não+encontrada';

const DIFFICULTIES = {
  'Fácil': 4,
  'Médio': 6,
  'Difícil': 8,
  'Impossível': 12,
};
const difficultyLevels = Object.keys(DIFFICULTIES);

export default function QuebraCabecaComMinhaPretinha() {
  const { width, height } = useWindowSize();
  const [difficulty, setDifficulty] = useState(1);
  const cols = useMemo(() => DIFFICULTIES[difficultyLevels[difficulty]], [difficulty]);
  const rows = cols;

  const [imageSrc, setImageSrc] = useState(DEFAULT_LOCAL_IMAGE);
  const [imageLoaded, setImageLoaded] = useState(false);

  const [pieces, setPieces] = useState([]);
  const [board, setBoard] = useState([]);
  const [win, setWin] = useState(false);
  const [time, setTime] = useState(0);
  const [isActive, setIsActive] = useState(false);
  const [draggingPiece, setDraggingPiece] = useState(null);

  const boardSize = Math.min(width * 0.8, height * 0.7, 600);
  const pieceSizeInTray = Math.min(72, boardSize / 6);

  // preload image on client and fallback if missing
  useEffect(() => {
    if (typeof window === 'undefined') return;
    let mounted = true;
    setImageLoaded(false);
    const img = new Image();
    img.src = imageSrc;
    img.onload = () => { if (mounted) setImageLoaded(true); };
    img.onerror = () => {
      if (!mounted) return;
      if (imageSrc !== FALLBACK_IMAGE) setImageSrc(FALLBACK_IMAGE);
      else setImageLoaded(true);
    };
    return () => { mounted = false; };
  }, [imageSrc]);

  const shuffle = useCallback(() => {
    setIsActive(true);
    setWin(false);
    setTime(0);

    const totalPieces = cols * rows;
    const newPieces = Array.from({ length: totalPieces }, (_, i) => ({ id: i, rotation: Math.floor(Math.random() * 4) * 90 }));

    // Fisher–Yates shuffle
    for (let i = newPieces.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [newPieces[i], newPieces[j]] = [newPieces[j], newPieces[i]];
    }

    setPieces(newPieces);
    setBoard(new Array(totalPieces).fill(null));
  }, [cols, rows]);

  // auto-shuffle when difficulty or image is ready
  useEffect(() => {
    if (!imageLoaded) return;
    shuffle();
  }, [difficulty, imageLoaded, shuffle]);

  // timer
  useEffect(() => {
    let interval = null;
    if (isActive && !win && typeof window !== 'undefined') {
      interval = setInterval(() => setTime(t => t + 1), 1000);
    }
    return () => clearInterval(interval);
  }, [isActive, win]);

  useEffect(() => {
    checkWinCondition();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [board, cols, rows]);

  function checkWinCondition() {
    if (board.includes(null) || board.length !== cols * rows) return;
    const isWin = board.every((p, idx) => p && p.id === idx && p.rotation % 360 === 0);
    if (isWin) {
      setWin(true);
      setIsActive(false);
    }
  }

  function handleDragStart(e, piece) {
    setDraggingPiece(piece);
    try { e.dataTransfer?.setData('text/plain', String(piece.id)); } catch (err) {}
  }

  function handleDropOnBoard(e, index) {
    e.preventDefault();
    if (!draggingPiece) return;
    const current = board[index];
    const newBoard = [...board];
    newBoard[index] = draggingPiece;
    if (current) setPieces(p => [...p, current]);
    setPieces(p => p.filter(x => x.id !== draggingPiece.id));
    setBoard(newBoard);
    setDraggingPiece(null);
  }

  function handleDropOnTray(e) {
    e.preventDefault();
    if (!draggingPiece) return;
    const pos = board.findIndex(p => p && p.id === draggingPiece.id);
    if (pos !== -1) {
      const newBoard = [...board];
      newBoard[pos] = null;
      setBoard(newBoard);
      setPieces(p => [...p, draggingPiece]);
    }
    setDraggingPiece(null);
  }

  function handlePieceClick(piece) {
    const rotate = p => ({ ...p, rotation: (p.rotation + 90) % 360 });
    const pos = board.findIndex(p => p && p.id === piece.id);
    if (pos !== -1) {
      setBoard(b => b.map((p, i) => (i === pos ? rotate(p) : p)));
    } else {
      setPieces(ps => ps.map(p => (p.id === piece.id ? rotate(p) : p)));
    }
  }

  function formatTime(seconds) {
    const m = Math.floor(seconds / 60).toString().padStart(2, '0');
    const s = (seconds % 60).toString().padStart(2, '0');
    return `${m}:${s}`;
  }

  function solve() {
    const solved = Array.from({ length: cols * rows }, (_, i) => ({ id: i, rotation: 0 }));
    setBoard(solved);
    setPieces([]);
    setWin(true);
    setIsActive(false);
  }

  function handleUpload(file) {
    if (!file) return;
    const reader = new FileReader();
    reader.onload = (ev) => {
      setImageSrc(String(ev.target.result));
    };
    reader.readAsDataURL(file);
  }

  // Helpers used during render to keep JSX simple
  const gridStyle = { width: boardSize, height: boardSize, gridTemplateColumns: `repeat(${cols}, 1fr)`, gridTemplateRows: `repeat(${rows}, 1fr)` };

  return (
    <div className="w-full min-h-screen bg-gradient-to-br from-green-50 to-green-100 text-gray-800 flex flex-col items-center p-4 overflow-hidden">
      {win && typeof window !== 'undefined' && (
        <Confetti width={width} height={height} recycle={false} numberOfPieces={400} />
      )}

      <motion.h1 initial={{ opacity: 0, y: -18 }} animate={{ opacity: 1, y: 0 }} className="text-3xl font-bold text-green-700 my-4 flex items-center gap-2">
        <Heart className="text-green-500" /> Quebra-cabeça com a minha pretinha {'<3'} <Heart className="text-green-500" />
      </motion.h1>

      <Card className="w-full max-w-5xl bg-white/70 backdrop-blur-sm shadow-xl border-green-200">
        <CardHeader className="flex flex-row items-center justify-between flex-wrap gap-4">
          <div className="flex flex-col">
            <CardTitle className="flex items-center gap-2 text-green-700">
              <BrainCircuit /> Nível: {difficultyLevels[difficulty]} ({cols}x{cols})
            </CardTitle>

            <div className="w-48 mt-2">
              <Slider
                min={0}
                max={difficultyLevels.length - 1}
                step={1}
                value={[difficulty]}
                onValueChange={(value) => setDifficulty(value[0])}
              />
            </div>
          </div>

          <div className="flex items-center gap-2 text-2xl font-semibold text-green-600">
            <Clock size={24} />
            <span>{formatTime(time)}</span>
          </div>

          <div className="flex gap-2">
            <Dialog>
              <DialogTrigger asChild>
                <Button variant="outline">
                  <ImageIcon className="mr-2 h-4 w-4" />Ver foto
                </Button>
              </DialogTrigger>

              <DialogContent className="max-w-xl">
                <DialogHeader>
                  <DialogTitle>Pré-visualização</DialogTitle>
                </DialogHeader>
                <img src={imageSrc} alt="Preview" className="rounded-lg w-full object-cover" />
                {!imageLoaded && <p className="mt-2 text-sm text-gray-500">Carregando imagem...</p>}
              </DialogContent>
            </Dialog>

            {/* upload/select image button */}
            <Button asChild>
              <label className="cursor-pointer flex items-center">
                <Upload className="mr-2 h-4 w-4" />Selecionar Imagem
                <input type="file" accept="image/*" className="hidden" onChange={(e) => handleUpload(e.target.files?.[0])} />
              </label>
            </Button>

            <Button onClick={shuffle}><Shuffle className="mr-2 h-4 w-4" />Embaralhar</Button>
            <Button onClick={solve} variant="destructive"><Check className="mr-2 h-4 w-4" />Resolver</Button>
          </div>
        </CardHeader>

        <CardContent className="flex flex-col lg:flex-row items-start justify-center gap-8 pt-6">
          <motion.div initial={{ scale: 0.9, opacity: 0 }} animate={{ scale: 1, opacity: 1 }} className="grid shadow-inner bg-green-50/70 rounded-lg overflow-hidden border-2 border-green-200" style={gridStyle} onDragOver={(e) => e.preventDefault()}>
            {board.map((p, idx) => {
              const cellKey = idx;
              if (!p) return (
                <div key={cellKey} className="relative outline outline-1 outline-green-200/50" onDragOver={(e) => e.preventDefault()} onDrop={(e) => handleDropOnBoard(e, idx)} />
              );

              const denomX = cols > 1 ? (cols - 1) : 1;
              const denomY = rows > 1 ? (rows - 1) : 1;
              const posX = ((p.id % cols) / denomX) * 100;
              const posY = ((Math.floor(p.id / cols)) / denomY) * 100;
              const pieceStyle = {
                width: '100%',
                height: '100%',
                backgroundImage: `url(${imageSrc})`,
                backgroundSize: `${cols * 100}% ${rows * 100}%`,
                backgroundPosition: `${posX}% ${posY}%`,
                transform: `rotate(${p.rotation}deg)`,
              };

              return (
                <div key={cellKey} className="relative outline outline-1 outline-green-200/50" onDragOver={(e) => e.preventDefault()} onDrop={(e) => handleDropOnBoard(e, idx)}>
                  <div draggable={!win} onDragStart={(e) => handleDragStart(e, p)} style={pieceStyle} onClick={() => handlePieceClick(p)} />
                </div>
              );
            })}
          </motion.div>

          <div className="w-full lg:w-80 flex-shrink-0">
            <Card className="bg-white/50 h-[30rem] lg:h-full" onDragOver={(e) => e.preventDefault()} onDrop={handleDropOnTray}>
              <CardHeader>
                <CardTitle className="text-green-700 flex items-center gap-2"><PuzzleIcon /> Peças Soltas</CardTitle>
              </CardHeader>

              <CardContent className="p-2 h-[calc(100%-4rem)] overflow-y-auto">
                <div className="flex flex-wrap gap-2 justify-center">
                  {pieces.map((p) => {
                    const denomX = cols > 1 ? (cols - 1) : 1;
                    const denomY = rows > 1 ? (rows - 1) : 1;
                    const posX = ((p.id % cols) / denomX) * 100;
                    const posY = ((Math.floor(p.id / cols)) / denomY) * 100;
                    const styleObj = {
                      width: pieceSizeInTray,
                      height: pieceSizeInTray,
                      backgroundImage: `url(${imageSrc})`,
                      backgroundSize: `${cols * 100}% ${rows * 100}%`,
                      backgroundPosition: `${posX}% ${posY}%`,
                      borderRadius: 8,
                    };

                    return (
                      <div key={p.id} draggable={!win} onDragStart={(e) => handleDragStart(e, p)} onClick={() => handlePieceClick(p)} style={styleObj} />
                    );
                  })}
                </div>
              </CardContent>
            </Card>
          </div>
        </CardContent>
      </Card>

      <AnimatePresence>
        {win && (
          <motion.div initial={{ opacity: 0, scale: 0.8 }} animate={{ opacity: 1, scale: 1 }} exit={{ opacity: 0, scale: 0.8 }} className="absolute inset-0 bg-black/30 flex items-center justify-center z-50" onClick={() => setWin(false)}>
            <Card className="bg-white p-6 text-center shadow-2xl border-4 border-green-400">
              <CardTitle className="text-2xl font-bold text-green-700 mb-3">Parabéns meu amor! 💖</CardTitle>
              <CardContent>
                <p className="text-lg">Você resolveu o quebra-cabeça!</p>
                <p className="text-lg mt-2">Seu tempo: <strong className="text-green-700">{formatTime(time)}</strong></p>
                <div className="mt-4">
                  <Button onClick={() => { setWin(false); shuffle(); }}>Jogar de Novo</Button>
                </div>
              </CardContent>
            </Card>
          </motion.div>
        )}
      </AnimatePresence>
    </div>
  );
}

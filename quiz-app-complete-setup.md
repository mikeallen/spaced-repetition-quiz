# Quiz App - Complete Setup Guide

## Quick Setup Instructions

### Step 1: Create Project Directory

```bash
mkdir spaced-repetition-quiz
cd spaced-repetition-quiz
```

### Step 2: Initialize React Project

```bash
npm create vite@latest . -- --template react
npm install
npm install lucide-react
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

### Step 3: Configure Tailwind

Create/update `tailwind.config.js`:

```javascript
/** @type {import('tailwindcss').Config} */
export default {
  content: [
    "./index.html",
    "./src/**/*.{js,ts,jsx,tsx}",
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

### Step 4: Update CSS

Replace `src/index.css` with:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

### Step 5: Copy App Code

Replace `src/App.jsx` with the complete app code (provided below).

### Step 6: Run the App

```bash
npm run dev
```

---

## Complete App Code (src/App.jsx)

Copy this entire code block into `src/App.jsx`:

```jsx
import React, { useState } from 'react';
import { Upload, Check, X, RotateCcw, HelpCircle } from 'lucide-react';

export default function QuizApp() {
  const [data, setData] = useState([]);
  const [headers, setHeaders] = useState([]);
  const [currentQuestion, setCurrentQuestion] = useState(null);
  const [score, setScore] = useState(0);
  const [attempts, setAttempts] = useState(0);
  const [userAnswer, setUserAnswer] = useState('');
  const [showResult, setShowResult] = useState(false);
  const [isCorrect, setIsCorrect] = useState(false);
  const [needsConfirmation, setNeedsConfirmation] = useState(false);
  const [ignoredQuestions, setIgnoredQuestions] = useState([]);
  const [selectedChoice, setSelectedChoice] = useState(null);
  const [showExplanation, setShowExplanation] = useState(false);
  const [explanation, setExplanation] = useState('');
  const [loadingExplanation, setLoadingExplanation] = useState(false);
  const [explainTopic, setExplainTopic] = useState('');
  const [questionSchedule, setQuestionSchedule] = useState({});
  const [questionCounts, setQuestionCounts] = useState({});

  const cleanValue = (value) => {
    if (!value) return '';
    return value.replace(/^["'\s]+|["'\s]+$/g, '');
  };

  const splitByComma = (text) => {
    if (!text) return [];
    const result = [];
    let current = '';
    let parenDepth = 0;
    
    for (let i = 0; i < text.length; i++) {
      const char = text[i];
      if (char === '(') {
        parenDepth++;
        current += char;
      } else if (char === ')') {
        parenDepth--;
        current += char;
      } else if (char === ',' && parenDepth === 0) {
        if (current.trim()) {
          result.push(cleanValue(current.trim()));
        }
        current = '';
      } else {
        current += char;
      }
    }
    
    if (current.trim()) {
      result.push(cleanValue(current.trim()));
    }
    
    return result;
  };

  const handleFileUpload = (e) => {
    const file = e.target.files[0];
    if (!file) return;

    const reader = new FileReader();
    reader.onload = (event) => {
      const text = event.target.result;
      const lines = text.split('\n').filter(line => line.trim());
      
      if (lines.length < 2) {
        alert('File needs at least a header row and one data row');
        return;
      }

      const headerRow = lines[0].split('\t').map(h => h.trim());
      const dataRows = lines.slice(1).map(line => {
        const values = line.split('\t').map(v => v.trim());
        const row = {};
        headerRow.forEach((header, idx) => {
          row[header] = values[idx] || '';
        });
        return row;
      });

      setHeaders(headerRow);
      setData(dataRows);
      setScore(0);
      setAttempts(0);
      setIgnoredQuestions([]);
      setQuestionSchedule({});
      setQuestionCounts({});
      generateQuestion(dataRows, headerRow);
    };
    reader.readAsText(file);
  };

  const generateQuestion = (dataSet, headerSet) => {
    if (dataSet.length === 0 || headerSet.length < 2) return;

    const questionableHeaders = headerSet.filter(h => h.toLowerCase() !== 'importance');
    if (questionableHeaders.length < 2) return;

    const entityField = questionableHeaders[0];
    const entityName = entityField.toLowerCase();
    const detailFields = questionableHeaders.slice(1);

    const now = Date.now();
    const isMultipleChoice = Math.random() < 0.8;

    const possibleQuestions = [];

    for (const row of dataSet) {
      for (const detailField of detailFields) {
        const entityValue = cleanValue(row[entityField]);
        const detailValue = row[detailField];
        
        if (!detailValue || detailValue === '""' || detailValue.trim() === '') continue;

        const forwardKey = `${entityValue}:${detailField}`;
        const forwardSchedule = questionSchedule[forwardKey];
        const forwardIgnored = ignoredQuestions.includes(forwardKey);
        
        if (!forwardIgnored) {
          const forwardDue = forwardSchedule ? forwardSchedule.nextTime : 0;
          const isReady = forwardDue <= now;
          
          possibleQuestions.push({
            type: 'forward',
            row,
            detailField,
            key: forwardKey,
            isImportant: row.importance === '1' || row.Importance === '1',
            dueTime: forwardDue,
            isReady: isReady,
            isDue: isReady && forwardSchedule
          });
        }

        const detailValues = detailValue.includes(',') ? splitByComma(detailValue) : [cleanValue(detailValue)];
        for (const dVal of detailValues) {
          if (!dVal) continue;
          const reverseKey = `${detailField}:${dVal}`;
          const reverseSchedule = questionSchedule[reverseKey];
          const reverseIgnored = ignoredQuestions.includes(reverseKey);
          
          if (!reverseIgnored) {
            const reverseDue = reverseSchedule ? reverseSchedule.nextTime : 0;
            const isReady = reverseDue <= now;
            
            possibleQuestions.push({
              type: 'reverse',
              row,
              detailField,
              detailValue: dVal,
              key: reverseKey,
              isImportant: row.importance === '1' || row.Importance === '1',
              dueTime: reverseDue,
              isReady: isReady,
              isDue: isReady && reverseSchedule
            });
          }
        }
      }
    }

    if (possibleQuestions.length === 0) {
      console.log('No questions available');
      return;
    }

    const readyQuestions = possibleQuestions.filter(q => q.isReady);
    const dueQuestions = readyQuestions.filter(q => q.isDue);
    const newQuestions = readyQuestions.filter(q => !q.isDue);

    if (readyQuestions.length === 0) {
      console.log('No ready questions - all scheduled for later');
      return;
    }

    let selectedQuestion;
    
    if (dueQuestions.length > 0 && Math.random() < 0.9) {
      const importantDue = dueQuestions.filter(q => q.isImportant);
      const otherDue = dueQuestions.filter(q => !q.isImportant);
      
      if (importantDue.length > 0 && (otherDue.length === 0 || Math.random() < 0.8)) {
        selectedQuestion = importantDue[Math.floor(Math.random() * importantDue.length)];
      } else if (otherDue.length > 0) {
        selectedQuestion = otherDue[Math.floor(Math.random() * otherDue.length)];
      } else {
        selectedQuestion = dueQuestions[Math.floor(Math.random() * dueQuestions.length)];
      }
    } else if (newQuestions.length > 0) {
      const importantNew = newQuestions.filter(q => q.isImportant);
      const otherNew = newQuestions.filter(q => !q.isImportant);
      
      if (importantNew.length > 0 && (otherNew.length === 0 || Math.random() < 0.8)) {
        selectedQuestion = importantNew[Math.floor(Math.random() * importantNew.length)];
      } else if (otherNew.length > 0) {
        selectedQuestion = otherNew[Math.floor(Math.random() * otherNew.length)];
      } else {
        selectedQuestion = newQuestions[Math.floor(Math.random() * newQuestions.length)];
      }
    } else {
      selectedQuestion = readyQuestions[Math.floor(Math.random() * readyQuestions.length)];
    }

    let questionPrompt, answerValues, choices = [], choiceOrigins = [];
    let questionContext = '';
    let quotedText = '';
    const isReverse = selectedQuestion.type === 'reverse';

    if (isReverse) {
      const detailValue = selectedQuestion.detailValue;
      
      answerValues = [];
      for (const row of dataSet) {
        const rowValue = row[selectedQuestion.detailField];
        if (rowValue && rowValue !== '""') {
          const values = rowValue.includes(',') ? splitByComma(rowValue) : [cleanValue(rowValue)];
          if (values.some(v => v.toLowerCase() === detailValue.toLowerCase())) {
            answerValues.push(cleanValue(row[entityField]));
          }
        }
      }
      
      questionPrompt = `What ${entityName} has the ${selectedQuestion.detailField} of "${detailValue}"?`;
      questionContext = detailValue;
      quotedText = detailValue;
      
      if (isMultipleChoice && answerValues.length > 0) {
        const wrongChoices = dataSet
          .map(row => cleanValue(row[entityField]))
          .filter(ent => !answerValues.includes(ent) && ent);
        
        const selectedWrong = [];
        while (selectedWrong.length < 4 && wrongChoices.length > 0) {
          const idx = Math.floor(Math.random() * wrongChoices.length);
          selectedWrong.push(wrongChoices[idx]);
          wrongChoices.splice(idx, 1);
        }
        
        choices = [...selectedWrong, answerValues[0]].sort(() => Math.random() - 0.5);
        if (!choices.some(c => answerValues.includes(c))) {
          choices[0] = answerValues[0];
        }
        
        choiceOrigins = choices.map(choice => ({
          choice,
          origin: answerValues.includes(choice) ? 'Correct answer' : `${entityField} name`
        }));
      }
    } else {
      const entityValue = cleanValue(selectedQuestion.row[entityField]);
      const answerValue = selectedQuestion.row[selectedQuestion.detailField];
      
      questionPrompt = `What is the ${selectedQuestion.detailField} of ${entityValue}?`;
      
      if (answerValue.includes(',')) {
        answerValues = splitByComma(answerValue).filter(v => v);
      } else {
        answerValues = [cleanValue(answerValue)];
      }
      
      questionContext = entityValue;
      
      if (isMultipleChoice) {
        const allValues = new Set();
        for (const row of dataSet) {
          const val = row[selectedQuestion.detailField];
          if (val && val !== '""' && val.trim() !== '') {
            if (val.includes(',')) {
              splitByComma(val).forEach(v => v && allValues.add(v));
            } else {
              const cleaned = cleanValue(val);
              if (cleaned) allValues.add(cleaned);
            }
          }
        }
        
        const wrongChoices = Array.from(allValues).filter(v => !answerValues.includes(v));
        const selectedWrong = [];
        while (selectedWrong.length < 4 && wrongChoices.length > 0) {
          const idx = Math.floor(Math.random() * wrongChoices.length);
          selectedWrong.push(wrongChoices[idx]);
          wrongChoices.splice(idx, 1);
        }
        
        choices = [...selectedWrong, answerValues[0]].sort(() => Math.random() - 0.5);
        if (!choices.some(c => answerValues.includes(c))) {
          choices[0] = answerValues[0];
        }
        
        choiceOrigins = choices.map(choice => {
          if (answerValues.includes(choice)) {
            return { choice, origin: `Correct answer - ${selectedQuestion.detailField} of ${entityValue}` };
          }
          for (const row of dataSet) {
            const val = row[selectedQuestion.detailField];
            if (val && val !== '""') {
              const values = val.includes(',') ? splitByComma(val) : [cleanValue(val)];
              if (values.includes(choice)) {
                return { choice, origin: `${selectedQuestion.detailField} of ${cleanValue(row[entityField])}` };
              }
            }
          }
          return { choice, origin: 'Unknown' };
        });
      }
    }

    const quoteMatch = questionPrompt.match(/"([^"]+)"/);
    if (quoteMatch) quotedText = quoteMatch[1];

    const currentCount = questionCounts[selectedQuestion.key] || 0;
    setQuestionCounts(prev => ({ ...prev, [selectedQuestion.key]: currentCount + 1 }));
    
    setCurrentQuestion({
      questionPrompt,
      answers: answerValues,
      key: selectedQuestion.key,
      isMultipleChoice,
      choices,
      choiceOrigins,
      questionContext,
      isReverse,
      quotedText,
      timesAsked: currentCount + 1
    });
    setUserAnswer('');
    setSelectedChoice(null);
    setShowResult(false);
    setNeedsConfirmation(false);
    setShowExplanation(false);
  };

  const levenshteinDistance = (str1, str2) => {
    const s1 = str1.toLowerCase();
    const s2 = str2.toLowerCase();
    const dp = Array(s1.length + 1).fill(null).map(() => Array(s2.length + 1).fill(0));
    
    for (let i = 0; i <= s1.length; i++) dp[i][0] = i;
    for (let j = 0; j <= s2.length; j++) dp[0][j] = j;
    
    for (let i = 1; i <= s1.length; i++) {
      for (let j = 1; j <= s2.length; j++) {
        if (s1[i - 1] === s2[j - 1]) {
          dp[i][j] = dp[i - 1][j - 1];
        } else {
          dp[i][j] = Math.min(dp[i - 1][j - 1], dp[i - 1][j], dp[i][j - 1]) + 1;
        }
      }
    }
    return dp[s1.length][s2.length];
  };

  const isCloseMatch = (userAns, correctAns) => {
    const distance = levenshteinDistance(userAns, correctAns);
    const maxLen = Math.max(userAns.length, correctAns.length);
    return distance <= Math.max(2, Math.floor(maxLen * 0.2));
  };

  const updateSchedule = (questionKey, correct) => {
    const now = Date.now();
    const currentSchedule = questionSchedule[questionKey];
    
    let nextTime;
    if (!correct) {
      nextTime = now + (5 * 60 * 1000);
    } else {
      if (!currentSchedule || !currentSchedule.correctCount) {
        nextTime = now + (24 * 60 * 60 * 1000);
      } else if (currentSchedule.correctCount === 1) {
        nextTime = now + (3 * 24 * 60 * 60 * 1000);
      } else {
        nextTime = now + (7 * 24 * 60 * 60 * 1000);
      }
    }
    
    setQuestionSchedule(prev => ({
      ...prev,
      [questionKey]: {
        nextTime,
        correctCount: correct ? ((prev[questionKey]?.correctCount || 0) + 1) : 0
      }
    }));
  };

  const handleSubmit = () => {
    if (!currentQuestion) return;
    
    const answer = currentQuestion.isMultipleChoice ? selectedChoice : userAnswer.trim();
    if (!answer) return;

    const normalizedAnswer = answer.toLowerCase();
    const exactMatch = currentQuestion.answers.some(ans => ans.toLowerCase() === normalizedAnswer);

    if (exactMatch) {
      setIsCorrect(true);
      setShowResult(true);
      setNeedsConfirmation(false);
      setAttempts(attempts + 1);
      setScore(score + 1);
      updateSchedule(currentQuestion.key, true);
      return;
    }

    if (!currentQuestion.isMultipleChoice) {
      const closeMatch = currentQuestion.answers.some(ans => isCloseMatch(answer, ans));
      if (closeMatch) {
        setNeedsConfirmation(true);
        setShowResult(true);
        setAttempts(attempts + 1);
        return;
      }
    }

    setIsCorrect(false);
    setShowResult(true);
    setNeedsConfirmation(false);
    setAttempts(attempts + 1);
    updateSchedule(currentQuestion.key, false);
  };

  const handleDontKnow = () => {
    setIsCorrect(false);
    setShowResult(true);
    setNeedsConfirmation(false);
    setAttempts(attempts + 1);
    updateSchedule(currentQuestion.key, false);
  };

  const handleConfirmCorrect = () => {
    setIsCorrect(true);
    setNeedsConfirmation(false);
    setScore(score + 1);
    updateSchedule(currentQuestion.key, true);
  };

  const handleConfirmWrong = () => {
    setIsCorrect(false);
    setNeedsConfirmation(false);
    updateSchedule(currentQuestion.key, false);
  };

  const handleMarkCorrect = () => {
    setScore(score + 1);
    setIsCorrect(true);
    updateSchedule(currentQuestion.key, true);
  };

  const handleNext = () => {
    generateQuestion(data, headers);
  };

  const handleIgnore = () => {
    setIgnoredQuestions(prev => [...prev, currentQuestion.key]);
    generateQuestion(data, headers);
  };

  const handleReset = () => {
    setScore(0);
    setAttempts(0);
    setIgnoredQuestions([]);
    setQuestionSchedule({});
    setQuestionCounts({});
    if (data.length > 0) {
      generateQuestion(data, headers);
    }
  };

  const handleExplain = async (topic = null) => {
    if (!currentQuestion) return;
    
    let explainText;
    if (topic) {
      explainText = topic;
    } else if (currentQuestion.isReverse) {
      explainText = currentQuestion.questionContext;
    } else {
      explainText = currentQuestion.answers[0];
    }
    
    setExplainTopic(explainText);
    setLoadingExplanation(true);
    setShowExplanation(true);
    
    try {
      const response = await fetch('https://api.anthropic.com/v1/messages', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          model: 'claude-sonnet-4-20250514',
          max_tokens: 1000,
          messages: [{
            role: 'user',
            content: `Provide a brief 2-3 sentence explanation about "${explainText}". Be concise and informative.`
          }]
        })
      });
      
      const data = await response.json();
      const text = data.content
        .filter(item => item.type === 'text')
        .map(item => item.text)
        .join('\n');
      
      setExplanation(text || 'Unable to load explanation.');
    } catch (error) {
      setExplanation('Unable to load explanation. Please try again.');
    } finally {
      setLoadingExplanation(false);
    }
  };

  if (data.length === 0) {
    return (
      <div className="min-h-screen bg-gradient-to-br from-red-50 to-white flex items-center justify-center p-4">
        <div className="bg-white rounded-lg shadow-lg p-8 max-w-md w-full">
          <h1 className="text-3xl font-bold text-red-600 mb-6 text-center">
            Quiz App
          </h1>
          <div className="border-2 border-dashed border-gray-300 rounded-lg p-8 text-center hover:border-red-400 transition-colors">
            <Upload className="w-12 h-12 text-gray-400 mx-auto mb-4" />
            <label className="cursor-pointer">
              <span className="text-blue-600 hover:text-blue-700 font-semibold">
                Upload TSV file
              </span>
              <input
                type="file"
                accept=".tsv,.txt"
                onChange={handleFileUpload}
                className="hidden"
              />
            </label>
            <p className="text-gray-500 text-sm mt-2">
              Tab-separated file with data
            </p>
          </div>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gradient-to-br from-red-50 to-white flex items-center justify-center p-4">
      <div className="bg-white rounded-lg shadow-lg p-8 max-w-2xl w-full">
        <div className="flex justify-between items-center mb-6">
          <h1 className="text-3xl font-bold text-red-600">Quiz</h1>
          <button
            onClick={handleReset}
            className="flex items-center gap-2 px-4 py-2 bg-gray-100 hover:bg-gray-200 rounded-lg transition-colors"
          >
            <RotateCcw className="w-4 h-4" />
            Reset
          </button>
        </div>

        <div className="bg-red-50 rounded-lg p-4 mb-6">
          <div className="flex justify-between text-lg">
            <span className="font-semibold">Score: {score}/{attempts}</span>
            <span className="text-gray-600">
              {attempts > 0 ? `${Math.round((score/attempts) * 100)}%` : '0%'}
            </span>
          </div>
        </div>

        {currentQuestion && (
          <div className="space-y-6">
            <div className="bg-gray-50 rounded-lg p-6">
              <div className="flex items-start justify-between">
                <p className="text-2xl font-bold text-gray-900 flex-1">
                  {currentQuestion.questionPrompt}
                </p>
                <span className="ml-3 text-xs text-gray-500 bg-gray-200 px-2 py-1 rounded">
                  #{currentQuestion.timesAsked - 1}
                </span>
              </div>
            </div>

            {!showResult ? (
              currentQuestion.isMultipleChoice ? (
                <div className="space-y-4">
                  <div className="space-y-2">
                    {currentQuestion.choices.map((choice, idx) => (
                      <button
                        key={idx}
                        onClick={() => setSelectedChoice(choice)}
                        className={`w-full text-left px-4 py-3 rounded-lg border-2 transition-colors ${
                          selectedChoice === choice
                            ? 'border-red-500 bg-red-50'
                            : 'border-gray-300 hover:border-red-300'
                        }`}
                      >
                        <span className="font-semibold mr-2">{String.fromCharCode(65 + idx)}.</span>
                        {choice}
                      </button>
                    ))}
                  </div>
                  <div className="flex gap-3">
                    <button
                      onClick={handleSubmit}
                      disabled={!selectedChoice}
                      className="flex-1 bg-red-600 hover:bg-red-700 disabled:bg-gray-300 disabled:cursor-not-allowed text-white font-semibold py-3 rounded-lg transition-colors"
                    >
                      Submit Answer
                    </button>
                    <button
                      onClick={handleDontKnow}
                      className="flex items-center justify-center gap-2 px-6 bg-gray-200 hover:bg-gray-300 text-gray-700 font-semibold py-3 rounded-lg transition-colors"
                    >
                      <HelpCircle className="w-5 h-5" />
                      Don't Know
                    </button>
                  </div>
                </div>
              ) : (
                <div className="space-y-4">
                  <input
                    type="text"
                    value={userAnswer}
                    onChange={(e) => setUserAnswer(e.target.value)}
                    onKeyPress={(e) => {
                      if (e.key === 'Enter') {
                        handleSubmit();
                      }
                    }}
                    placeholder="Type your answer..."
                    className="w-full px-4 py-3 border-2 border-gray-300 rounded-lg focus:border-red-500 focus:outline-none text-lg"
                    autoFocus
                  />
                  <div className="flex gap-3">
                    <button
                      onClick={handleSubmit}
                      className="flex-1 bg-red-600 hover:bg-red-700 text-white font-semibold py-3 rounded-lg transition-colors"
                    >
                      Submit Answer
                    </button>
                    <button
                      onClick={handleDontKnow}
                      className="flex items-center justify-center gap-2 px-6 bg-gray-200 hover:bg-gray-300 text-gray-700 font-semibold py-3 rounded-lg transition-colors"
                    >
                      <HelpCircle className="w-5 h-5" />
                      Don't Know
                    </button>
                  </div>
                </div>
              )
            ) : needsConfirmation ? (
              <div className="space-y-4">
                <div className="rounded-lg p-6 bg-yellow-50 border-2 border-yellow-400">
                  <p className="text-lg font-semibold text-yellow-900 mb-3">
                    Close match detected!
                  </p>
                  <div className="space-y-2 mb-4">
                    <p className="text-gray-700">
                      Your answer: <span className="font-semibold">{userAnswer}</span>
                    </p>
                    <p className="text-gray-700">
                      Expected: <span className="font-semibold">{currentQuestion.answers.join(' or ')}</span>
                    </p>
                  </div>
                  <p className="text-gray-600 mb-4">Was your answer correct?</p>
                  <div className="flex gap-3">
                    <button
                      onClick={handleConfirmCorrect}
                      className="flex-1 bg-green-600 hover:bg-green-700 text-white font-semibold py-3 rounded-lg transition-colors flex items-center justify-center gap-2"
                    >
                      <Check className="w-5 h-5" />
                      Yes, Correct
                    </button>
                    <button
                      onClick={handleConfirmWrong}
                      className="flex-1 bg-red-600 hover:bg-red-700 text-white font-semibold py-3 rounded-lg transition-colors flex items-center justify-center gap-2"
                    >
                      <X className="w-5 h-5" />
                      No, Wrong
                    </button>
                  </div>
                </div>
              </div>
            ) : (
              <div className="space-y-4">
                <div className={`rounded-lg p-6 ${isCorrect ? 'bg-green-50 border-2 border-green-500' : 'bg-red-50 border-2 border-red-500'}`}>
                  <div className="flex items-center gap-3 mb-3">
                    {isCorrect ? (
                      <Check className="w-8 h-8 text-green-600" />
                    ) : (
                      <X className="w-8 h-8 text-red-600" />
                    )}
                    <span className={`text-xl font-bold ${isCorrect ? 'text-green-600' : 'text-red-600'}`}>
                      {isCorrect ? 'Correct!' : 'Incorrect'}
                    </span>
                  </div>
                  {isCorrect && currentQuestion.answers.length > 1 && (
                    <div className="mt-3">
                      <p className="text-gray-700">
                        All correct answers: {currentQuestion.answers.map((ans, idx) => (
                          <span key={idx}>
                            {idx > 0 && ', '}
                            <button
                              onClick={() => handleExplain(ans)}
                              className="font-semibold text-green-700 hover:text-green-800 underline"
                            >
                              {ans}
                            </button>
                          </span>
                        ))}
                      </p>
                    </div>
                  )}
                  {!isCorrect && (
                    <div className="space-y-1">
                      {(userAnswer || selectedChoice) && (
                        <p className="text-gray-700">
                          Your answer: <span className="font-semibold">{currentQuestion.isMultipleChoice ? selectedChoice : userAnswer}</span>
                        </p>
                      )}
                      <p className="text-gray-700">
                        Correct answer{currentQuestion.answers.length > 1 ? 's' : ''}: {currentQuestion.answers.map((ans, idx) => (
                          <span key={idx}>
                            {idx > 0 && ' or '}
                            <button
                              onClick={() => handleExplain(ans)}
                              className="font-semibold text-green-700 hover:text-green-800 underline"
                            >
                              {ans}
                            </button>
                          </span>
                        ))}
                      </p>
                      {currentQuestion.quotedText && (
                        <p className="text-gray-700 mt-2">
                          <button
                            onClick={() => handleExplain(currentQuestion.quotedText)}
                            className="text-blue-600 hover:text-blue-800 underline"
                          >
                            What is "{currentQuestion.quotedText}"?
                          </button>
                        </p>
                      )}
                    </div>
                  )}
                  
                  {currentQuestion.isMultipleChoice && currentQuestion.choiceOrigins && currentQuestion.choiceOrigins.length > 0 && (
                    <div className="mt-4 pt-4 border-t border-gray-300">
                      <p className="text-sm font-semibold text-gray-700 mb-2">Multiple choice options:</p>
                      <div className="grid grid-cols-2 gap-2">
                        {currentQuestion.choiceOrigins.map((item, idx) => {
                          const isCorrectAnswer = currentQuestion.answers.includes(item.choice);
                          return (
                            <button
                              key={idx}
                              onClick={() => handleExplain(item.choice)}
                              className={`text-sm p-2 rounded text-left transition-colors ${
                                isCorrectAnswer 
                                  ? 'bg-green-100 border border-green-300 hover:bg-green-200' 
                                  : 'bg-gray-50 hover:bg-gray-100'
                              }`}
                            >
                              <div className="font-medium text-gray-900">{item.choice}</div>
                              <div className="text-gray-600 text-xs mt-1">{item.origin}</div>
                            </button>
                          );
                        })}
                      </div>
                    </div>
                  )}
                </div>
                <div className="flex gap-2 flex-wrap">
                  <button
                    onClick={handleNext}
                    onKeyPress={(e) => {
                      if (e.key === 'Enter') handleNext();
                    }}
                    className="flex-1 min-w-32 bg-blue-600 hover:bg-blue-700 text-white font-semibold py-3 rounded-lg transition-colors"
                    autoFocus
                  >
                    Next Question
                  </button>
                  {!isCorrect && (
                    <>
                      <button
                        onClick={handleMarkCorrect}
                        className="px-4 bg-green-600 hover:bg-green-700 text-white font-semibold py-3 rounded-lg transition-colors"
                      >
                        Correct
                      </button>
                      <button
                        onClick={handleIgnore}
                        className="px-4 bg-gray-500 hover:bg-gray-600 text-white font-semibold py-3 rounded-lg transition-colors"
                      >
                        Ignore
                      </button>
                    </>
                  )}
                </div>
              </div>
            )}
          </div>
        )}

        <div className="mt-6 text-center text-sm text-gray-500">
          {ignoredQuestions.length > 0 && `${ignoredQuestions.length} question${ignoredQuestions.length === 1 ? '' : 's'} ignored`}
        </div>

        {showExplanation && (
          <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4 z-50">
            <div className="bg-white rounded-lg shadow-xl p-6 max-w-lg w-full">
              <h3 className="text-xl font-bold text-gray-900 mb-4">
                About {explainTopic}
              </h3>
              {loadingExplanation ? (
                <div className="text-center py-8">
                  <div className="inline-block animate-spin rounded-full h-8 w-8 border-b-2 border-red-600"></div>
                  <p className="mt-2 text-gray-600">Loading explanation...</p>
                </div>
              ) : (
                <p className="text-gray-700 leading-relaxed mb-6">
                  {explanation}
                </p>
              )}
              <button
                onClick={() => setShowExplanation(false)}
                className="w-full bg-red-600 hover:bg-red-700 text-white font-semibold py-3 rounded-lg transition-colors"
              >
                Close
              </button>
            </div>
          </div>
        )}
      </div>
    </div>
  );
}
```

---

## Project Structure

After setup, your project should look like this:

```
spaced-repetition-quiz/
├── node_modules/
├── public/
├── src/
│   ├── App.jsx          # Main quiz app (code above)
│   ├── index.css        # Tailwind styles
│   └── main.jsx         # Entry point
├── .gitignore
├── index.html
├── package.json
├── tailwind.config.js
├── vite.config.js
└── README.md            # This file
```

---

## Development History

### Phase 1: Initial Development
- Created basic quiz structure with TSV file upload
- Implemented bidirectional questions (entity→detail and detail→entity)
- Added multiple choice (5 options) and free text modes

### Phase 2: Enhanced Question Logic
- Fixed comma-separated values with parentheses preservation
- Added importance weighting (80% importance=1, 20% importance=2)
- Implemented fuzzy matching for close answers with confirmation dialog
- Added "Don't Know" button

### Phase 3: Spaced Repetition System
- Implemented scheduling system:
  - Wrong answers: Review in 5 minutes
  - First correct: Review in 1 day
  - Second correct: Review in 3 days
  - Third+ correct: Review in 7 days
- Added question tracking with repeat counter (#0, #1, #2, etc.)
- Fixed priority system: 90% due questions (reviews), 10% new questions

### Phase 4: UI/UX Improvements
- Changed to 80% multiple choice, 20% free text
- Added AI-powered explanations via Anthropic API
- Made answers and multiple choice options clickable for explanations
- Added "What is [quoted text]?" links for terms in questions
- Implemented "Correct" override button
- Added "Ignore" button to permanently skip questions
- Always show repeat counter (even for first-time questions)

### Phase 5: Generic Data Support
- Made app work with any entity type (not just prefectures)
- Uses first column name dynamically in questions
- Generic titles ("Quiz" instead of "Japan Quiz")

### Phase 6: Bug Fixes & Optimization
- Fixed quote stripping in TSV parsing
- Ensured at least one correct answer in multiple choice
- Improved question generation algorithm to properly track and schedule reviews
- Added proper categorization of due vs new questions
- Implemented randomization within importance tiers

---

## TSV File Format

```tsv
Entity	Field1	Field2	Field3	importance
Value1	Data1	Data2	Data3	1
Value2	Data1	Data2	Data3	2
```

### Rules
- First row: Headers (field names)
- First column: Main entity (e.g., Prefecture, Country, City)
- `importance` column (optional): `1` for high priority, `2` for lower
- Empty values: Use `""` or leave blank
- Multiple values: Comma-separated (e.g., `Tokyo, Edo`)
- Parentheses: Commas inside are preserved (e.g., `Tokyo (Kanto), Osaka`)

---

## Features

✅ **Spaced Repetition** - Intelligent review scheduling
✅ **Multiple Choice & Free Text** - 80/20 split
✅ **Bidirectional Questions** - Tests both directions
✅ **Fuzzy Matching** - Accepts close answers
✅ **AI Explanations** - Powered by Claude API
✅ **Question Tracking** - See how many times asked
✅ **Ignore System** - Skip questions permanently
✅ **Override Corrections** - Mark wrong answers as correct
✅ **Importance Weighting** - Focus on priority items
✅ **Generic Data Support** - Works with any entity type

---

## Running with Claude CLI

Once you have the project set up:

```bash
# Start development server
npm run dev

# Work with Claude CLI for further development
claude code review src/App.jsx
claude code "add feature X to the quiz"
```

---

## Git Setup (Optional)

```bash
git init
git add .
git commit -m "Initial commit: Spaced repetition quiz app"
```

---

## Notes

- All data stays in browser memory (no persistence across sessions)
- AI explanations require internet connection
- Press Enter to advance to next question after seeing results
- Multiple choice options show where each answer came from

---

## License

Free to use and modify for personal and educational purposes.
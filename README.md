'use client'
import { Button } from '@/components/ui/button'
import { api } from '@/convex/_generated/api'
import { AIModel } from '@/services/GlobalServices'
import { CoachingExpert } from '@/services/Options'
import { UserButton } from '@stackframe/stack'
import { useQuery } from 'convex/react'
import Image from 'next/image'
import { useParams } from 'next/navigation'
import React, { useEffect, useRef, useState } from 'react'
import ChatBox from './__components/ChatBox'

function DiscussionRoom() {
  const { roomid } = useParams()
  const DiscussionRoomData = useQuery(api.DiscussionRoom.GetDiscussionRoom, { id: roomid })
  const [expert, setExpert] = useState(null)
  const [enableMic, setEnableMic] = useState(false)
  const [transcript, setTranscript] = useState([
    {
      role: 'user',
      content: 'hi'
    },
    {
      role: 'assistant',
      content: 'hello'
    }
  ])
  const [loading, setLoading] = useState(false)
  const recognition = useRef(null)

  useEffect(() => {
    if (DiscussionRoomData) {
      const Expert = CoachingExpert.find(item => item.name === DiscussionRoomData.expertName)
      setExpert(Expert)
    }
  }, [DiscussionRoomData])

  useEffect(() => {
    console.log("Updated transcript:", transcript)
  }, [transcript])

  const startRecognition = () => {
    if (!recognition.current) {
      recognition.current = new webkitSpeechRecognition();
      recognition.current.continuous = true;
      recognition.current.interimResults = true;

      recognition.current.onstart = () => {
        console.log("Speech recognition started");
      };

      recognition.current.onend = () => {
        console.log("Speech recognition ended");
        // Restart recognition if mic is still enabled
        if (enableMic) {
          console.log("Restarting speech recognition...");
          recognition.current.start();
        }
      };

      recognition.current.onresult = async (event) => {
        console.log("Speech result received:", event.results);
        let finalTranscript = '';
        for (let i = event.resultIndex; i < event.results.length; i++) {
          const text = event.results[i][0].transcript;
          console.log(`Result ${i}:`, text, "isFinal:", event.results[i].isFinal);
          if (event.results[i].isFinal) {
            finalTranscript += text + ' ';
          }
        }
        if (finalTranscript.trim()) {
          console.log("Final transcript:", finalTranscript);
          setTranscript(prev => [...prev, { role: 'user', content: finalTranscript.trim() }]);
        }
      };

      recognition.current.onerror = (event) => {
        console.error("Speech recognition error:", event.error);
        if (event.error === 'no-speech') {
          // Restart on no-speech error
          if (enableMic) {
            console.log("No speech detected, restarting...");
            recognition.current.start();
          }
        }
      };
    }
  };

  const connectToServer = async () => {
    try {
      console.log("Starting speech recognition setup...");

      if (!('webkitSpeechRecognition' in window)) {
        console.error("Speech recognition not supported in this browser");
        alert('Speech recognition is not supported in this browser. Please use Chrome.');
        return;
      }

      startRecognition();
      recognition.current.start();
      setEnableMic(true);
    } catch (error) {
      console.error("Error in speech recognition setup:", error);
      alert('Error starting speech recognition. Please try again.');
    }
  };

  const disconnect = (e) => {
    e.preventDefault();
    console.log("Disconnecting speech recognition...");
    setEnableMic(false);
    
    if (recognition.current) {
      recognition.current.stop();
      recognition.current = null;
      console.log("Speech recognition stopped");
    }
  };

  useEffect(() => {
    async function fetchData() {
      const lastMessage = transcript[transcript.length - 1];
      if (lastMessage.role === 'user') {
        setLoading(true);
        try {
          const aiResp = await AIModel(
            DiscussionRoomData?.topic || '', 
            DiscussionRoomData?.CoachingOption || '', 
            lastMessage.content
          );
          console.log("AI response:", aiResp);
          if (aiResp) {
            // Format the response to match the demo structure
            const formattedResponse = {
              role: 'assistant',
              content: aiResp
            };
            setTranscript(prev => [...prev, formattedResponse]);
          }
        } catch (error) {
          console.error("Error getting AI response:", error);
        } finally {
          setLoading(false);
        }
      }
    }
    if (DiscussionRoomData) {
      fetchData();
    }
  }, [transcript, DiscussionRoomData]);

  return (
    <div className='-mt-12'>
      <div className='mt-5 grid grid-cols-1 lg:grid-cols-3 gap-10'>
        <div className='lg:col-span-2'>
          <h2 className='mt-[-32px] pb-[10px] pl-2 font-bold text-lg'>
            {DiscussionRoomData?.CoachingOption}
          </h2>
          <div className='h-[60vh] bg-secondary rounded-4xl flex flex-col items-center justify-center relative'>
            {expert && (
              <Image
                src={expert.avatar || '/default-avatar.png'}
                alt={expert.name || 'Expert'}
                width={200}
                height={200}
                className='h-[80px] w-[80px] rounded-full object-cover animate-pulse'
              />
            )}
            <h2 className='text-gray-500'>{expert?.name}</h2>
            <div className='p-5 bg-gray-200 px-10 rounded-lg absolute right-8 bottom-8'>
              <UserButton />
            </div>
          </div>
          <div className='text-center mt-5'>
            {!enableMic ?
              <Button onClick={connectToServer}>Connect</Button>
              :
              <Button onClick={disconnect} variant="destructive">Disconnect</Button>
            }
          </div>
        </div>
        <div className='lg:col-span-1'>
          <ChatBox transcript={transcript} />
          <h2 className='mt-4 text-gray-400 text-sm'>
            At the end of your conversation we will automatically generate feedback/notes from your conversation
          </h2>
        </div>
      </div>
    </div>
  )
}

export default DiscussionRoom

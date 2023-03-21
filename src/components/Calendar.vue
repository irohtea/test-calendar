<script setup>
import { ref, computed, reactive, onMounted } from 'vue';
import { onClickOutside } from '@vueuse/core'

import {startOfWeek, startOfMonth, endOfMonth, eachDayOfInterval, 
format, differenceInCalendarDays, 
subDays, endOfWeek, addDays, isToday, isWithinInterval,   } from 'date-fns';
import { zonedTimeToUtc, utcToZonedTime, formatInTimeZone } from 'date-fns-tz';


   const days = ['Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun']
   const currentDay = ref(new Date())
   const currentMonth = computed(() => format(currentDay.value, 'MMMM yyyy'))
   
   const dates = ref([])
   const generateDates = () => {
      const monthStart = startOfMonth(currentDay.value)
      const monthEnd = endOfMonth(currentDay.value)
      const daysInMonth = eachDayOfInterval({ start: monthStart, end: monthEnd })
      const firstDayOfWeek = startOfWeek(monthStart, { weekStartsOn: 1 })
      const leadingDays = differenceInCalendarDays(monthStart, firstDayOfWeek) > 0 ?
         eachDayOfInterval({ start: firstDayOfWeek, end: subDays(monthStart, 1) }) :
         []
      const lastDayOfWeek = endOfWeek(monthEnd, { weekStartsOn: 1 })
      const trailingDays = differenceInCalendarDays(lastDayOfWeek, monthEnd) > 0 ?
         eachDayOfInterval({ start: addDays(monthEnd, 1), end: lastDayOfWeek }) :
         []
      const allDays = [...leadingDays, ...daysInMonth, ...trailingDays]
      dates.value =  allDays.map(date => reactive({
         fullDate: format(date, 'yyyy-MM-dd'),
         date: format(date, 'd'),
         isToday: isToday(date),
         isInMonth: isWithinInterval(date, { start: monthStart, end: monthEnd }),
         events: []
      }))
   }
   onMounted(() => {
      generateDates()
   })
   const prevMonth = () => {
      currentDay.value = new Date(currentDay.value.getFullYear(), currentDay.value.getMonth() - 1, 1)
      generateDates()
   }
   const nextMonth = () => {
      currentDay.value = new Date(currentDay.value.getFullYear(), currentDay.value.getMonth() + 1, 1)
      generateDates()
   }
   const isDialogOpen = ref(false)
   const toggleDialog = () => {
      isDialogOpen.value = !isDialogOpen.value

   }
   const modal = ref(null)
   onClickOutside(modal, () => {
      isDialogOpen.value = !isDialogOpen.value

   })
   const newEvent = ref({
      title: '',
      start: '',
      end: '',
      time: '',
   })
   const clickedDate = ref('')
   const openModal = (date) => {
      clickedDate.value = date
      toggleDialog()
   }
   const createEvent = () => {
      const pickedDate = dates.value.find((date) => date.fullDate === clickedDate.value);
      if (pickedDate) {
         const kyivTimeZone = 'Europe/Kyiv'
         const startTime = utcToZonedTime(zonedTimeToUtc(`${pickedDate.fullDate}T${newEvent.value.start}`, kyivTimeZone), kyivTimeZone);
         const endTime = utcToZonedTime(zonedTimeToUtc(`${pickedDate.fullDate}T${newEvent.value.end}`, kyivTimeZone), kyivTimeZone);
         if(newEvent.value.title != '') {
            pickedDate.events.push({
               name: newEvent.value.title,
               start: formatInTimeZone(startTime, kyivTimeZone, 'yyyy-MM-dd HH:mm'),
               end: formatInTimeZone(endTime, kyivTimeZone, 'yyyy-MM-dd HH:mm'),
            });
         }
        
      }
      newEvent.value = {
         title: '',
         start: '',
         end: '',
      };
      toggleDialog();
};
</script>

<template>
   <div class="border border-gray-700 rounded shadow-2xl">
      <div class="p-5 flex items-center justify-between bg-sky-500">
         <button class="p-2 rounded bg-sky-600 transition hover:bg-sky-700" @click="prevMonth">
            <svg class="w-6 h-6 dark:text-white" fill="none" stroke="currentColor" stroke-width="1.5" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg" aria-hidden="true">
               <path stroke-linecap="round" stroke-linejoin="round" d="M6.75 15.75L3 12m0 0l3.75-3.75M3 12h18"></path>
            </svg>
         </button>
         <h2 class="text-lg text-white font-bold">{{ currentMonth }}</h2>
         <button class="p-2 rounded bg-sky-600 transition hover:bg-sky-700" @click="nextMonth">
            <svg class="w-6 h-6 dark:text-white" fill="none" stroke="currentColor" stroke-width="1.5" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg" aria-hidden="true">
               <path stroke-linecap="round" stroke-linejoin="round" d="M17.25 8.25L21 12m0 0l-3.75 3.75M21 12H3"></path>
            </svg>
         </button>
      </div>
      <div class="pt-2 bg-slate-800">
         <div class="pb-2 grid grid-cols-7 justify-items-center">
            <div class="text-white font-bold" v-for="day in days" :key="day">
               {{ day }}
            </div>
         </div>
         <div class="grid grid-cols-7 justify-items-center">
            <div
            @click="openModal(date.fullDate)" 
            class="h-24 w-full flex justify-center border border-gray-700 text-white cursor-pointer transition hover:bg-sky-700" 
            :class="[
               date.isInMonth ? '' : 'text-gray-500 bg-slate-900 pointer-events-none', 
               !date.isToday ? '' : 'text-sky-400 bg-emerald-400 -font-semibold',]"
               v-for="(date, index) in dates" :key="index">
               <div class="flex flex-col items-center ">
                  <div class="pt-1" >{{ date.date }}</div>
                  <div>
                     <div v-for="(event, index) in date.events" :key="index" >
                        <div class="p-1 bg-indigo-600 rounded font-light text-sm">
                           <div class="text-ellipsis whitespace-nowrap w-20 overflow-hidden"> {{ event.name }} </div>
                           {{ formatInTimeZone(event.start, 'Europe/London', 'HH:mm') }} - {{ formatInTimeZone(event.end, 'Europe/London', 'HH:mm') }}
                        </div>
                     </div>
                  </div>
               </div>
            </div>
         </div>
      </div>
   </div>

   <Teleport to="#modal">
      <transition name="fade">
         <div 
         v-if="isDialogOpen"
         class="fixed top-0 left-0 z-50 w-full h-full bg-black bg-opacity-50 flex justify-center items-center" >
            <div class=" p-6 bg-white" @click="toggleDialog" ref="modal">
               <form @submit.prevent="createEvent" @click.stop>
                  <div class="flex flex-col gap-3">
                     <div class="flex flex-col gap-1">
                        <label class="text-sm" for="title">Event start</label>
                        <input class="border-b border-indigo-500 " required placeholder="Interview" name="title" type="text" v-model="newEvent.title">
                     </div>
                     <div class="flex flex-col gap-3">
                        <label class="text-sm" for="title">Event start:</label>
                        <input type="time" name="start" v-model="newEvent.start">
                     </div>
                     <div class="flex flex-col gap-3">
                        <label class="text-sm" for="title">Event end:</label>
                        <input type="time" name="end" v-model="newEvent.end">
                     </div>
                     <button class="p-2 rounded text-white bg-sky-600 transition hover:bg-sky-700" type="submit">Add event</button>
                  </div>
               </form>
            </div>
         </div>
      </transition>
   </Teleport>
</template>

<style scoped>
   .fade-enter-active,
   .fade-leave-active {
      transition: all 0.25s ease 0s;
   }

   .fade-enter-from,
   .fade-leave-to {
      opacity: 0;
      transform: scale(1.1);
   }
</style>
<template>
  <div class="gated-content-schedule-page">
    <div v-if="endDate" class="date-filter">
      <button @click.stop="backOneWeek" role="button" :disabled="disablePrevDateButton"
              aria-label="previous date">
        <i class="fa fa-angle-left"></i>
      </button>
      <span class="date" v-cloak>
        {{ startDate | month_short }} {{ startDate | day }} -
        {{ endDate | month_short }} {{ endDate | day }}
      </span>
      <button @click.stop="forwardOneWeek" role="button" aria-label="next date">
        <i class="fa fa-angle-right"></i>
      </button>
    </div>
    <div v-if="loading" class="text-center">
      <Spinner></Spinner>
    </div>
    <template v-else>
      <div v-if="error">Error loading</div>
      <div v-else class="calendar">
        <div class="mobile">
          <template v-for="(day, index) in listing">
            <div v-if="upcomingEvents(day) > 0" :key="index" class="day"
                 :class="{'collapsed': collapses[index]}">
              <div class="header"
                   @click.stop="collapse(index);">
                <div class="caption">
                  {{ day.date | weekday }},&nbsp;
                  {{ day.date | month }} {{ day.date | day }}
                </div>
                <div class="count">
                  {{ upcomingEvents(day) }}
                  {{ upcomingEvents(day) > 1 ? 'Events' : 'Event' }}
                </div>
                <button role="button" class="day-collapse" aria-label="collapse day">
                  <i class="fa fa-minus" aria-hidden="true"></i>
                </button>
                <button role="button" class="day-expand" aria-label="expand day">
                  <i class="fa fa-plus" aria-hidden="true"></i>
                </button>
              </div>
              <div class="items">
                <template v-for="items in day.hourSlots">
                  <template v-if="typeof items !== 'undefined'">
                    <template v-for="event in items">
                      <ScheduleEventCard v-if="event !== null" :event="event" :key="event.id"/>
                    </template>
                  </template>
                </template>
              </div>
            </div>
          </template>
        </div>
        <div class="desktop">
          <div class="slot dates">
            <div class="hour-row">
              <div v-for="(day, index) in listing" :key="index" class="event-cell date"
                   :class="{'active': currentDay(day.date)}">
                <div class="weekday">{{ day.date | weekday }}</div>
                <div class="date">{{ day.date | month }} {{ day.date | day }}</div>
              </div>
            </div>
          </div>
          <template v-for="(eventsCount, hour) in hours">
            <div v-if="typeof eventsCount !== 'undefined'" class="slot" :key="hour">
              <div class="caption">
                <template v-for="(day, index) in listing">
                  <div class="hour-card" :class="{'active': currentDay(day.date)}" :key="index">
                    <template v-if="outputHours(index)">{{ hour | hour }}</template>
                  </div>
                </template>
              </div>
              <div v-for="slotPlace in eventsCount" :key="slotPlace" class="hour-row">
                <div v-for="(day, index) in listing" :key="index" class="event-cell">
                  <ScheduleEventCard
                    v-if="typeof day.hourSlots[hour] !== 'undefined'
                      && typeof day.hourSlots[hour][slotPlace - 1] !== 'undefined'"
                    :event="day.hourSlots[hour][slotPlace - 1]"/>
                </div>
              </div>
            </div>
          </template>
        </div>
        <div v-if="listingIsEmpty" class="empty-listing">No events scheduled for this week.</div>
      </div>
    </template>
  </div>
</template>

<script>
import dayjs from 'dayjs';
import client from '@/client';
import Spinner from '@/components/Spinner.vue';
import ScheduleEventCard from '@/components/event/ScheduleEventCard.vue';

export default {
  name: 'SchedulePage',
  components: {
    Spinner,
    ScheduleEventCard,
  },
  props: {
    msg: {
      String,
      default: 'Events not found.',
    },
  },
  data() {
    return {
      loading: true,
      error: false,
      listing: null,
      hours: null,
      collapses: null,
      startDate: null,
      endDate: null,
    };
  },
  watch: {
    $route: 'initStartDate',
    startDate: 'load',
  },
  async mounted() {
    this.initStartDate();
  },
  computed: {
    listingIsEmpty() {
      let count = 0;
      this.listing.forEach((day) => {
        day.hourSlots.forEach((slot) => { count += slot.length; });
      });
      return count === 0;
    },
    disablePrevDateButton() {
      return this.startDate.isBefore(dayjs());
    },
  },
  methods: {
    async load() {
      this.loading = true;
      this.endDate = this.startDate.endOf('week');

      client({
        url: '/virtual-y/api/events',
        method: 'get',
        params: {
          type: 'all',
          start_date: this.startDate.toISOString(),
          end_date: this.endDate.toISOString(),
        },
      })
        .then((response) => {
          this.listing = [];
          this.hours = [];
          this.collapses = [];
          for (let i = 0; i < 7; i += 1) {
            this.listing[i] = {
              date: this.startDate.add(i, 'day'),
              hourSlots: [],
            };
            this.collapses[i] = true;
          }
          response.data.forEach((event) => {
            const start = this.$dayjs.date(event.date.value);
            const day = start.day();
            const hour = start.hour();
            if (typeof this.listing[day].hourSlots[hour] === 'undefined') {
              this.listing[day].hourSlots[hour] = [];
            }
            this.listing[day].hourSlots[hour].push(event);
            if (typeof this.hours[hour] === 'undefined') {
              this.hours[hour] = [];
            }
            if (typeof this.hours[hour][day] === 'undefined') {
              this.hours[hour][day] = 1;
            } else {
              this.hours[hour][day] += 1;
            }
          });

          this.hours.forEach((eventsCount, hour) => {
            this.hours[hour] = eventsCount.reduce((a, b) => Math.max(a, b));
          });

          let found = false;
          this.listing.forEach((day, index) => {
            if (!found && this.upcomingEvents(day) > 0) {
              this.collapses[index] = false;
              found = true;
            }
          });

          this.loading = false;
        })
        .catch((error) => {
          this.error = true;
          this.loading = false;
          console.error(error);
        });
    },
    currentDay(date) {
      return dayjs().startOf('day').isSame(date, 'day');
    },
    outputHours(index) {
      const endOfWeek = dayjs().endOf('week');
      const day = this.listing[index].date;
      return (day.isAfter(endOfWeek) && index === 3) || this.currentDay(day);
    },
    backOneWeek() {
      this.startDate = this.startDate.subtract(1, 'week');
      this.updateRoute();
    },
    forwardOneWeek() {
      this.startDate = this.startDate.add(1, 'week');
      this.updateRoute();
    },
    upcomingEvents(day) {
      let count = 0;
      day.hourSlots.forEach((slot) => {
        slot.forEach((event) => {
          if (this.$dayjs.date(event.date.end_value).isAfter(dayjs())) {
            count += 1;
          }
        });
      });
      return count;
    },
    collapse(index) {
      this.collapses[index] = !this.collapses[index];
      this.$forceUpdate();
    },
    initStartDate() {
      let date = dayjs();
      if (this.$route.query.startDate && date.isBefore(dayjs.unix(this.$route.query.startDate))) {
        date = dayjs.unix(this.$route.query.startDate);
      }
      this.startDate = date.startOf('week');
      this.updateRoute();
    },
    updateRoute() {
      const query = {
        ...this.$route.query,
        startDate: this.startDate.unix(),
      };
      if (this.startDate.isBefore(dayjs())) {
        delete query.startDate;
      }
      if (Object.entries(this.$route.query).toString() !== Object.entries(query).toString()) {
        this.$router.push({ query });
      }
    },
  },
};
</script>

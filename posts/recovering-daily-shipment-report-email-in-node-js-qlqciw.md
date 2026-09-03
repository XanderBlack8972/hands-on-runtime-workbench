# Recovering Daily Shipment Report Email in Node.js with Cron and a Queue

Use one cron job to create daily report jobs, then let a message queue deliver each subscriber's email independently. The deciding constraint is operational recovery: after a crash, the system must show which shipment updates were sent, which are still pending, and which can be retried without sending duplicates.

This is still a simple architecture. Cron owns time. The queue owns fan-out and retries. A small SaaS doesn't need a distributed scheduling platform just to email a daily shipment report, but putting the entire subscriber loop inside one scheduled process makes a partial failure hard to resume. The revenue-per-hour choice is to outsource that undifferentiated delivery state to a queue while keeping report generation in ordinary application code.

## Govern each delivery, not each run

A cron-only design looks reasonable at first. At 08:00, one Node.js process queries every subscriber, renders each report, sends each email, and exits. The crontab format is built for this kind of recurring trigger, and the schedule is easy to inspect.

The smallest useful boundary is therefore one scheduled batch and one durable job per subscriber. The scheduler records a stable daily batch key such as `shipment-report:2026-08-12`, creates jobs with stable delivery keys, and finishes. Workers claim those jobs separately. An acknowledgement is sent only after the email provider accepts the request and the application records the outcome. RabbitMQ documents the underlying acknowledgement distinction directly: a consumer acknowledgement tells the broker that a delivery was processed, while a publisher confirm covers the publisher-to-broker side.

This boundary also prevents slow report rendering or a temporary downstream limit from holding the scheduler open. More important for a one-person operation, it gives the support screen a useful answer: this subscriber's report is pending, claimed, or sent. Shipping that visibility earns more than building a clever scheduler.

## Migrate one interrupted shipment batch

The trouble begins between recipients. Suppose the process has 600 subscriber reports to send and stops after recipient 417. Rerunning the whole command risks duplicate email for the first 417; doing nothing loses the rest. A single `lastRunAt` field can't describe that split. The operator needs a batch record for the logical day and 600 delivery records beneath it, with 417 outcomes recorded and 183 jobs still eligible for work. This isn't a complaint about cron. It is a mismatch between one time trigger and hundreds of independently recoverable delivery units, and it stays a mismatch even if the original loop is fast on an ordinary day.

Recovery is the feature.

Now restart the worker. It claims one of the remaining delivery jobs, renders that subscriber's shipment update, sends it with the stable delivery key, records the provider's message identifier, and acknowledges the job. If execution stops before the acknowledgement, the queue can expose the job again. The worker checks the durable sent state before producing another effect. That sequence turns “did today's script run?” into the narrower questions support actually needs to answer.

## How should a Node.js SaaS decide between a cron job and message queue for daily report email?

Use both when one scheduled event fans out into deliveries that need separate retry and audit state. Use cron alone when the task is atomic, quick, and harmless to rerun as a whole. Use a queue without cron when an external event already creates the work, such as a carrier webhook that should immediately produce a shipment update.

The decision can be made from failure boundaries, not company size:

| Work shape | Simple default | Reason |
| --- | --- | --- |
| One daily aggregate, one internal recipient | Cron job | The whole operation can be retried together |
| One daily batch, many subscriber emails | Cron plus queue | Each delivery needs its own recovery state |
| Update triggered by an incoming event | Queue | The event, rather than the clock, creates work |
| Strict ordering across updates | Queue with an explicit ordering design | Concurrent workers can otherwise change delivery order |

There is a catch. A queue adds another durable system to operate, plus dead-letter handling, worker deployment, and job inspection. It is not suitable when a missed run can simply be rerun as one unit and duplicate execution has no user-visible effect. In that case, stick with cron and a database run record. Fewer moving parts means more time for the weekly product release.

Cron also has time semantics that deserve an explicit decision. A crontab schedule runs in the cron daemon's configured time zone, and daylight-saving changes can cause some local times to occur twice or not at all. For a daily customer report, schedule in a declared time zone, store the logical report date separately from the execution timestamp, and make the batch key unique. Don't infer the report date from whichever machine happens to run the worker.

## The smallest Node.js integration boundary

Keep the scheduler thin. It should decide which logical day is due and enqueue deterministic work. It should not send email itself.

The following TypeScript uses generic storage, queue, and mail interfaces so the architectural contract stays visible. The database implementation must enforce uniqueness for `batchId` and `deliveryId`; checking first and inserting later in application code would leave a race between two scheduler instances.

```ts
type ShipmentReportJob = {
  deliveryId: string;
  batchId: string;
  subscriberId: string;
  reportDate: string;
};

interface ReportStore {
  createBatchOnce(batchId: string, reportDate: string): Promise<boolean>;
  listSubscriberIds(): Promise<string[]>;
  createDeliveryOnce(job: ShipmentReportJob): Promise<boolean>;
  markSent(deliveryId: string, providerMessageId: string): Promise<void>;
  wasSent(deliveryId: string): Promise<boolean>;
}

interface JobQueue {
  publish(job: ShipmentReportJob): Promise<void>;
}

interface ShipmentReports {
  render(subscriberId: string, reportDate: string): Promise<string>;
}

interface Mailer {
  sendShipmentReport(input: {
    subscriberId: string;
    html: string;
    idempotencyKey: string;
  }): Promise<{ messageId: string }>;
}

export async function scheduleDailyReports(
  reportDate: string,
  store: ReportStore,
  queue: JobQueue,
): Promise<void> {
  const batchId = `shipment-report:${reportDate}`;
  const created = await store.createBatchOnce(batchId, reportDate);
  if (!created) return;

  const subscriberIds = await store.listSubscriberIds();
  for (const subscriberId of subscriberIds) {
    const deliveryId = `${batchId}:${subscriberId}`;
    const job = { deliveryId, batchId, subscriberId, reportDate };

    if (await store.createDeliveryOnce(job)) {
      await queue.publish(job);
    }
  }
}

export async function deliverDailyReport(
  job: ShipmentReportJob,
  store: ReportStore,
  reports: ShipmentReports,
  mailer: Mailer,
): Promise<void> {
  if (await store.wasSent(job.deliveryId)) return;

  const html = await reports.render(job.subscriberId, job.reportDate);
  const result = await mailer.sendShipmentReport({
    subscriberId: job.subscriberId,
    html,
    idempotencyKey: job.deliveryId,
  });

  await store.markSent(job.deliveryId, result.messageId);
}
```

Run `scheduleDailyReports` from a crontab entry that invokes a compiled script. The five-field expression below means minute 0, hour 8, every day of the month, every month, every day of the week. The Linux manual notes that cron examines entries every minute.

```bash
0 8 * * * cd /srv/app && node dist/schedule-daily-reports.js
```

One subtle gap remains between publishing a job and recording it. If the database insert succeeds but publishing doesn't, the delivery exists without a queued message. Do not hide that state. Add a small reconciliation command that republishes delivery records which remain pending and have no active claim. The same deterministic `deliveryId` makes repeated reconciliation safe at the application boundary.

Make it boring.

Likewise, an email request can succeed before `markSent` is committed. Exactly-once effects cannot be assumed merely because a queue acknowledges messages. Pass the stable delivery key to a mail system that supports idempotent requests, or accept that a rare duplicate is possible and make it visible to support. I'm not sure every mail provider preserves an idempotency key for the same duration, so that retention window must be verified in the provider's current documentation before relying on it.

## Evaluate the release by interrupting it

The useful tests interrupt the workflow at its boundaries. Start the scheduler twice for the same report date and assert that only one batch and one delivery row per subscriber exist. Stop a worker after rendering but before sending; the job should return to pending. Stop it after the send response but before the database update; the test should document whether provider-level idempotency prevents a duplicate.

Then test a poison job: one subscriber has malformed template data, while every other subscriber still receives a report. Cap automatic attempts, preserve the final error classification with the delivery record, and move exhausted work out of the hot retry path for manual inspection. A retry counter without the last error and next-attempt time is barely operational data.

Observability can stay modest. Track scheduled batches, pending deliveries, oldest pending age, attempts, and sent deliveries. Alert on age, not just counts; ten pending jobs for two minutes may be normal, while one pending job for twelve hours is a customer-support problem. Logs should carry `batchId`, `deliveryId`, and `subscriberId` so one report can be traced without searching by email address.

Deployment needs one more guard: workers should stop claiming new jobs, finish or release current work, and only then exit. Queue acknowledgements belong after the durable outcome. If a worker acknowledges before sending, a crash loses the report. If it never acknowledges after recording success, redelivery is expected, so the sent check and stable key must make that path uneventful.

## Let measured cost trigger the next change

At larger subscriber counts, page through subscribers instead of loading every ID into memory, and let the scheduler publish bounded chunks. Apply queue backpressure so a daily spike doesn't overwhelm the mail provider. Partition work only after measurements show a real bottleneck; partitions make ordering and rebalancing harder to reason about.

I would also separate report computation from email delivery if rendering becomes expensive. A batch stage can build immutable report artifacts, while delivery jobs reference them. That lets rendering and sending scale independently, but it also adds artifact retention and privacy decisions. Your mileage may vary: for a modest daily report, generating HTML in the worker is easier to ship and easier to delete.

The final choice is deliberately plain. Keep cron as the visible clock, use durable per-recipient jobs when partial recovery matters, and make idempotency and reconciliation part of the first implementation. Don't buy scheduling complexity before the failure boundary calls for it.

## References

- https://man7.org/linux/man-pages/man5/crontab.5.html
- https://www.rabbitmq.com/docs/confirms

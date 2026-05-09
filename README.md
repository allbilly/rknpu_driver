This repo is extracted from official rknpu-driver. 

https://github.com/airockchip/rknn-llm/tree/release-v1.1.0/rknpu-driver

# Known issue

- mainline rocket driver crash will auto reboot but rknpu_driver just hang

```
The actual 10-line fix for the thermal issue is a per-core watchdog timer that force-powers the NPU down if a job stalls. Here's what it looks like:
First, add a timer to the subcore struct (rknpu_drv.h):
struct rknpu_subcore_data {
    ...
    struct timer_list watchdog_timer;  // +1 line
};
In rknpu_job_next() (when HW starts processing), arm it:
    mod_timer(&subcore_data->watchdog_timer,
              jiffies + msecs_to_jiffies(10000));  // +1 line
In rknpu_job_done() (IRQ confirms completion), disarm:
    del_timer(&subcore_data->watchdog_timer);  // +1 line
Timer callback forces power-down:
void rknpu_watchdog_cb(struct timer_list *t) {                    // +4 lines
    struct rknpu_subcore_data *sd = from_timer(sd, t, watchdog_timer);
    pm_runtime_put_sync(sd->rknpu_dev->dev);
    rknpu_soft_reset(sd->rknpu_dev);
}
That's 7 lines — and it guarantees the NPU powers off within 10 seconds of a crash, regardless of what userspace does, without killing the rest of the system.
```

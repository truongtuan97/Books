# Code Review: Rest Time Blocks Feature

## Overall Assessment

**Status: ❌ Request Changes**

While the basic structure is in place, there are several critical bugs and missing validations that prevent the feature from meeting the requirements. The implementation needs significant fixes before it can be approved.

---

## Critical Issues

### 1. **Missing Route Handler** 🔴
**Location:** `config/routes.rb:20` → `app/controllers/api/v1/rest_time_blocks_controller.rb`

**Issue:** The route `GET /api/v1/workers/:worker_id/rest_time_blocks` is defined but the `worker_rest_time_blocks` method doesn't exist in the controller.

**Fix Required:** Either implement the method or remove the route if not needed.

---

### 2. **Incorrect Overlap Validation Logic** 🔴
**Location:** `app/models/rest_time_block.rb:11-17`

**Issue:** The `no_overlap_with_jobs` validation checks ALL jobs in the system, not just jobs assigned to the current worker. This is incorrect.

**Current Code:**
```ruby
def no_overlap_with_jobs
  overlapping_jobs = Job.where("start_date < ? AND end_date > ?", end_time, start_time)
  # ...
end
```

**Problems:**
- Doesn't filter by `worker_id`
- Doesn't check if jobs are actually assigned (could check unassigned jobs)
- The overlap condition logic appears incorrect (should check if ranges overlap, not just if job spans the rest block)

**Fix Required:**
```ruby
def no_overlap_with_jobs
  return unless worker_id && start_time && end_time
  
  overlapping_jobs = worker.jobs
    .where("start_date < ? AND end_date > ?", end_time, start_time)
  
  if overlapping_jobs.exists?
    errors.add(:base, "Rest time block overlaps with assigned jobs")
  end
end
```

**Better overlap logic:**
```ruby
def no_overlap_with_jobs
  return unless worker_id && start_time && end_time
  
  overlapping_jobs = worker.jobs
    .where("(start_date < ? AND end_date > ?) OR (start_date < ? AND end_date > ?) OR (start_date >= ? AND end_date <= ?)",
           end_time, start_time,  # Job starts before and ends after rest block
           start_time, end_time,  # Job starts after rest block starts but before it ends
           start_time, end_time)  # Job is completely within rest block
  
  if overlapping_jobs.exists?
    errors.add(:base, "Rest time block overlaps with assigned jobs")
  end
end
```

---

### 3. **Incorrect Daily Rest Time Validation** 🔴
**Location:** `app/models/rest_time_block.rb:19-29`

**Issue:** The `maximum_daily_rest_time` validation only checks if the current block exceeds 2 hours, but the requirement states: **"Total number of hours resting during the day can't exceed 2 hrs"**. This means we need to sum ALL rest blocks for the day, not just check the current one.

**Current Code:**
```ruby
def maximum_daily_rest_time
  duration = (end_time - start_time) / 1.hour
  if duration > 2
    errors.add(:base, "Cannot have more than 2 hours of rest time")
  end
end
```

**Fix Required:**
```ruby
def maximum_daily_rest_time
  return unless worker_id && start_time && end_time
  
  # Get all rest blocks for this worker on the same day
  same_day_blocks = worker.rest_time_blocks
    .where("DATE(start_time) = ? OR DATE(end_time) = ?", start_time.to_date, start_time.to_date)
    .where.not(id: id) # Exclude current record if updating
  
  # Calculate total duration including this block
  current_duration = (end_time - start_time) / 1.hour
  existing_duration = same_day_blocks.sum do |block|
    # Only count hours that are on the same day
    block_start = [block.start_time, start_time.to_date.beginning_of_day].max
    block_end = [block.end_time, start_time.to_date.end_of_day].min
    (block_end - block_start) / 1.hour
  end
  
  total_duration = current_duration + existing_duration
  
  if total_duration > 2
    errors.add(:base, "Total rest time for the day cannot exceed 2 hours. Current total: #{total_duration.round(2)} hours")
  end
end
```

**Note:** This logic needs careful handling for blocks that span multiple days.

---

### 4. **Missing Job Assignment Validation** 🔴
**Location:** `app/models/assignment.rb` or `app/controllers/api/v1/assignments_controller.rb`

**Issue:** The requirement states: **"Similarly, a job can't be assigned to a worker if it overlaps with existing rest blocks"**. This validation is completely missing.

**Fix Required:** Add validation to `Assignment` model:
```ruby
class Assignment < ApplicationRecord
  # ... existing code ...
  
  validate :no_overlap_with_rest_time_blocks
  
  private
  
  def no_overlap_with_rest_time_blocks
    return unless worker_id && job_id
    
    job = self.job
    return unless job.start_date && job.end_date
    
    overlapping_rest_blocks = worker.rest_time_blocks.where(
      "(start_time < ? AND end_time > ?) OR (start_time < ? AND end_time > ?) OR (start_time >= ? AND end_time <= ?)",
      job.end_date, job.start_date,  # Rest block spans job
      job.start_date, job.end_date,   # Rest block starts during job
      job.start_date, job.end_date    # Rest block is within job
    )
    
    if overlapping_rest_blocks.exists?
      errors.add(:base, "Job overlaps with worker's rest time blocks")
    end
  end
end
```

---

### 5. **Security Issue: Worker ID in Params** 🔴
**Location:** `app/controllers/api/v1/rest_time_blocks_controller.rb:15, 39`

**Issue:** The controller allows `worker_id` to be passed in params, which means a worker could create rest time blocks for other workers. Since workers can only manage their own rest blocks, this should use `current_worker` instead.

**Current Code:**
```ruby
def create
  @rest_time_block = RestTimeBlock.new(rest_time_block_params)
  # ...
end

def rest_time_block_params
  params.require(:rest_time_block).permit(:worker_id, :start_time, :end_time)
end
```

**Fix Required:**
```ruby
def create
  @rest_time_block = current_worker.rest_time_blocks.build(rest_time_block_params)
  # ...
end

def rest_time_block_params
  params.require(:rest_time_block).permit(:start_time, :end_time)
  # Remove :worker_id from permitted params
end
```

---

### 6. **Debug Code Left in Production** 🟡
**Location:** `app/models/rest_time_block.rb:23`

**Issue:** There's a `puts` statement that should be removed.

**Fix Required:** Remove line 23: `puts "duration: #{duration}"`

---

## Additional Issues

### 7. **Missing Time Validation** 🟡
**Location:** `app/models/rest_time_block.rb`

**Issue:** No validation that `end_time` is after `start_time`.

**Fix Required:**
```ruby
validate :end_time_after_start_time

private

def end_time_after_start_time
  return unless start_time && end_time
  
  if end_time <= start_time
    errors.add(:end_time, "must be after start_time")
  end
end
```

---

### 8. **Missing Authorization Check** 🟡
**Location:** `app/controllers/api/v1/rest_time_blocks_controller.rb:32-36`

**Issue:** The `set_rest_time_block` method doesn't verify that the rest time block belongs to `current_worker`. A worker could delete another worker's rest time blocks by guessing IDs.

**Fix Required:**
```ruby
def set_rest_time_block
  @rest_time_block = current_worker.rest_time_blocks.find(params[:id])
rescue ActiveRecord::RecordNotFound
  render json: { error: "Rest time block not found" }, status: :not_found
end
```

---

### 9. **Potential Rest Block Overlap** 🟡
**Location:** `app/models/rest_time_block.rb`

**Issue:** While not explicitly stated in requirements, it's reasonable to prevent rest time blocks from overlapping with each other for the same worker.

**Suggestion:** Consider adding validation to prevent overlapping rest blocks.

---

### 10. **Time Zone Handling** 🟡
**Location:** `app/models/rest_time_block.rb`

**Issue:** Jobs have a `time_zone` attribute, but rest time blocks don't. This could cause issues when comparing times. Consider whether rest time blocks should also have time zones or if all times should be normalized to a common timezone.

---

## Positive Aspects

✅ Good use of Rails conventions and structure  
✅ Proper use of associations (`belongs_to`, `has_many`)  
✅ Appropriate use of `before_action` for authorization  
✅ Transaction usage in `assignments_controller`  
✅ Clear separation of concerns  

---

## Summary

The implementation has the right structure but contains critical bugs that prevent it from meeting the requirements:

1. ❌ Overlap validation checks wrong jobs (all jobs vs. worker's jobs)
2. ❌ Daily rest time validation only checks current block, not total
3. ❌ Missing validation when assigning jobs (no check for rest block overlap)
4. ❌ Security issue: workers can create blocks for other workers
5. ❌ Missing route handler implementation
6. ⚠️ Debug code left in production
7. ⚠️ Missing basic validations (end_time > start_time)

**Recommendation:** Fix all critical issues before merging. The feature needs significant work to meet the stated requirements.


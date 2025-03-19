# Writeup

## Week 6 - Lab F

## Q1. Particles Collisions

### Solution
```rs
use rand::Rng;
use std::time::Instant;

#[derive(Debug, Copy, Clone)]
pub struct Particle{
    x: f32,
    y: f32,
    id: i32,
}

impl Particle{
    pub fn new(x_param: f32, y_param: f32, id_param:i32) -> Particle{
        Particle{
            x: x_param,
            y: y_param,
            id:id_param
        }
    }  
    fn Collide(&self, particle: Particle, mut counter:i32) -> bool
    {
        let x_diff= particle.x - self.x;
        let y_diff = particle.y - self.y;

        return x_diff*x_diff + y_diff*y_diff <= 0.25*0.25;
    }
}

#[derive(Debug, Clone)]
pub struct ParticleSystem{
    particles: Vec<Particle>,
}



impl ParticleSystem{
    pub fn new() -> ParticleSystem {
        ParticleSystem {
            particles: Vec::new(),
        }
    }
    fn StartMoving(&mut self, system:&ParticleSystem){
        let num_iteration = 85000;
        let thread_count = 10;
        let particles_per_thread = self.particles.len()/thread_count;
        let collision_counter = 0;
        println!("Moving {} particles {} times across {} threads", self.particles.len(), num_iteration, thread_count);
        let start_time = Instant::now();
        let mut pool = scoped_threadpool::Pool::new(thread_count as u32);
        for j in 0..num_iteration{
            pool.scoped(|scope|{
                for chunk in self.particles.chunks_mut(particles_per_thread)
                {
                    scope.execute(move || thread_main(chunk));
                }
            });
        }   
        collisionCheck(&system.clone(), collision_counter);
        let duration = Instant::now().duration_since(start_time).as_secs();
        println!("Took {} seconds for particles to move", duration);
    }
}



const PARTICLE_COUNT: usize = 100;
const PARTICLE_BOUNDS:(i32,i32) = (10, 10);
const PARTICLE_BOUNDS_HALF:(f32,f32) = (PARTICLE_BOUNDS.0 as f32 * 0.5,PARTICLE_BOUNDS.1 as f32 * 0.5);

fn main() {
    let mut rng = rand::rng();
    let mut particleSystem = ParticleSystem::new();
    let mut j = 0;
    for _i in 0..PARTICLE_COUNT{
        let x = rng.random_range(-PARTICLE_BOUNDS_HALF.0..PARTICLE_BOUNDS_HALF.0);
        let y = rng.random_range(-PARTICLE_BOUNDS_HALF.1..PARTICLE_BOUNDS_HALF.1);

        let particle = Particle::new(x,y,j);

        particleSystem.particles.push(particle);
        j = j + 1;
    }
    particleSystem.StartMoving(&particleSystem.clone());
}

pub fn thread_main(chunk: &mut [Particle])
{
    let chunk_size = chunk.len();
    let mut rng = rand::rng();

        for i in 0..chunk_size{
            let mut x = rng.random::<f32>();
            let mut y = rng.random::<f32>();
            let negative_X = rng.random_range(0..=1);
            let negative_Y = rng.random_range(0..=1);
            if negative_X == 1{
                x = -x;
            }
            if negative_Y == 1{
                y = -y;
            }
    
            chunk[i].x += x;
            chunk[i].y += y;
    
            if chunk[i].x < -PARTICLE_BOUNDS_HALF.0{
                chunk[i].x = -PARTICLE_BOUNDS_HALF.0;
            }
            else if chunk[i].x > PARTICLE_BOUNDS_HALF.0{
                chunk[i].x = PARTICLE_BOUNDS_HALF.0;
            }
    
            if chunk[i].y < -PARTICLE_BOUNDS_HALF.0{
                chunk[i].y = -PARTICLE_BOUNDS_HALF.1;
            }
            else if chunk[i].y > PARTICLE_BOUNDS_HALF.0{
                chunk[i].y = PARTICLE_BOUNDS_HALF.1;
            }
        }
}

pub fn collisionCheck(system:&ParticleSystem, mut counter:i32){
    let list_of_particles = system.particles.len();
    for i in 0..list_of_particles{
        for j in 0..list_of_particles{

            if system.particles[i].id == system.particles[j].id 
            {
                continue;
            }
            let collide = system.particles[i].Collide(system.particles[j],counter);
            if collide == true{
                counter+=1;
            }

        }
    }
    println!("Number of collisions : {}" , counter);
}

```
### Output

![image](https://github.com/user-attachments/assets/18797c38-ff42-4583-930f-d522bb59764d)

In this case, the output shows the number of collisions in the final iteration of movement, not the total number of collisions across all movements

### Reflection

Was locking required in the code?
In my code, I did not need to put any locking in the code, as we did not need to prevent the collision checker from trying to access information that was being used. The collision checker only started accessing the data to find the information needed after the movement was finished, hence there was no need to lock the data to thread.

When we split the collision counter into chunks and run them asyncronously as threads, there is no need to lock the data as we are borrowing the value of the list of particles, and we are not modifying the value in the borrowed data, hence there is no need to lock.

One way that the code could be futher optimized would be to split the data that we are checking against into further chunks, and running those comaprisons asynchronously.


## Q2. Particles Collisions Atomic

### Solution
```rs
use rand::Rng;
use std::time::Instant;
use std::sync::{Arc};
use std::sync::atomic::{AtomicUsize, Ordering};

#[derive(Debug, Copy, Clone)]
pub struct Particle{
    x: f32,
    y: f32,
    id: i32,
}

impl Particle{
    pub fn new(x_param: f32, y_param: f32, id_param:i32) -> Particle{
        Particle{
            x: x_param,
            y: y_param,
            id:id_param
        }
    }  
    fn Collide(&self, particle: Particle) -> bool
    {
        let x_diff= particle.x - self.x;
        let y_diff = particle.y - self.y;

        return x_diff*x_diff + y_diff*y_diff <= 0.25*0.25;
    }
}

#[derive(Debug, Clone)]
pub struct ParticleSystem{
    particles: Vec<Particle>,
    collisionCounter: Arc<AtomicUsize>,
}



impl ParticleSystem{
    pub fn new() -> ParticleSystem {
        ParticleSystem {
            particles: Vec::new(),
            collisionCounter: Arc::new(AtomicUsize::new(0))
        }
    }
    fn StartMoving(&mut self, system:&ParticleSystem){
        let num_iteration = 85000;
        let thread_count = 10;
        let particles_per_thread = self.particles.len()/thread_count;
        let collision_counter = 0;
        println!("Moving {} particles {} times across {} threads", self.particles.len(), num_iteration, thread_count);
        let start_time = Instant::now();
        let mut pool = scoped_threadpool::Pool::new(thread_count as u32);
        for j in 0..num_iteration{
            pool.scoped(|scope|{
                for chunk in self.particles.chunks_mut(particles_per_thread)
                {
                    scope.execute(move || thread_main(chunk));
                }
            });
        }   
        let duration = Instant::now().duration_since(start_time).as_secs();
        let particles_clone = self.particles.clone();
        let self_clone = self.clone();
        let coll_count_clone = Arc::clone(&self.collisionCounter);
        collisionCheck(&particles_clone, &coll_count_clone, self_clone);
        println!("Took {} seconds for particles to move", duration);
    }
}



const PARTICLE_COUNT: usize = 100;
const PARTICLE_BOUNDS:(i32,i32) = (10, 10);
const PARTICLE_BOUNDS_HALF:(f32,f32) = (PARTICLE_BOUNDS.0 as f32 * 0.5,PARTICLE_BOUNDS.1 as f32 * 0.5);

fn main() {
    let mut rng = rand::rng();
    let mut particleSystem = ParticleSystem::new();
    let mut j = 0;
    for _i in 0..PARTICLE_COUNT{
        let x = rng.random_range(-PARTICLE_BOUNDS_HALF.0..PARTICLE_BOUNDS_HALF.0);
        let y = rng.random_range(-PARTICLE_BOUNDS_HALF.1..PARTICLE_BOUNDS_HALF.1);

        let particle = Particle::new(x,y,j);

        particleSystem.particles.push(particle);
        j = j + 1;
    }
    particleSystem.StartMoving(&particleSystem.clone());
}

pub fn thread_main(chunk: &mut [Particle])
{
    let chunk_size = chunk.len();
    let mut rng = rand::rng();

        for i in 0..chunk_size{
            let mut x = rng.random::<f32>();
            let mut y = rng.random::<f32>();
            let negative_X = rng.random_range(0..=1);
            let negative_Y = rng.random_range(0..=1);
            if negative_X == 1{
                x = -x;
            }
            if negative_Y == 1{
                y = -y;
            }
    
            chunk[i].x += x;
            chunk[i].y += y;
    
            if chunk[i].x < -PARTICLE_BOUNDS_HALF.0{
                chunk[i].x = -PARTICLE_BOUNDS_HALF.0;
            }
            else if chunk[i].x > PARTICLE_BOUNDS_HALF.0{
                chunk[i].x = PARTICLE_BOUNDS_HALF.0;
            }
    
            if chunk[i].y < -PARTICLE_BOUNDS_HALF.0{
                chunk[i].y = -PARTICLE_BOUNDS_HALF.1;
            }
            else if chunk[i].y > PARTICLE_BOUNDS_HALF.0{
                chunk[i].y = PARTICLE_BOUNDS_HALF.1;
            }
        }
}

pub fn collisionCheck(mut list: &Vec<Particle>, coll_counter_clone: &AtomicUsize,mut system_clone:ParticleSystem){
    let list_of_particles = list.len();
    let thread_count = 10;
    let mut thread_id = 1;
    let part_per_thread = list_of_particles/thread_count;
 
    let mut pool = scoped_threadpool::Pool::new(thread_count as u32);
    pool.scoped(|scope|{
        for chunk in system_clone.particles.chunks_mut(part_per_thread)
        {
            scope.execute(move || Count(chunk,&list,&coll_counter_clone, thread_id));
            thread_id+=1;
        }
    });


    println!("Tot num of Collisions: {}", coll_counter_clone.load(Ordering::Relaxed));
}

fn Count(chunk: &[Particle],list:&Vec<Particle>, coll_counter_clone:&AtomicUsize, thread_id:i32){
    let chunk_list = chunk.len();
    let mut counter = 0;
    let list_of_particles = list.len();
    for i in 0..chunk_list{
        for j in 0..list_of_particles{

            if chunk[i].id == list[j].id 
            {
                continue;
            }
            let collide = chunk[i].Collide(list[j]);
            if collide == true{
                counter+=1;
                coll_counter_clone.fetch_add(1, Ordering::Relaxed);
            }

        }
    }
    println!("Number of collisions in thread {}: {}" ,thread_id, counter);
}

```
### Output

![image](https://github.com/user-attachments/assets/64bb447d-0733-4150-b81c-58ed63f63b8e)

The above output shows the number of collisions in the final iteration of movement, not the total number of collisions across all movements.



### Reflection

In this exercise, I have learnt more about the usage of atomics, and how they work.
In my code, the atomic counter works as a shared value in the original particle systems, and we are able to make changes to the original when we make changes to the references.
Atomics are able to allow us to make modifications and access data across multiple threads without data races, through various methods, such as ensuring memory syncronisation, allowing all threads to see changes made in one thread.
They also ensure that operations done on atomic variables either complete, or do not happen, ensuring that that there are no partial or incomplete writes from different threads. 

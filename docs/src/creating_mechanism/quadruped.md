# Loading a Mechanism via URDF
Another way to build a mechanism is to directly load it from a URDF file. We illustrate this with the A1 quadruped, defined in `DojoEnvironments`. 

![quadruped](../assets/pictures/quadruped.png)

Note that the [`Mechanism`](@ref) is created simply by passing the path to the URDF file to the constructor. Afterwards, additional features, such as contacts can be added. 

```julia
function get_quadruped(;
    timestep=0.01,
    input_scaling=timestep, 
    gravity=-9.81,
    urdf=:gazebo_a1, 
    springs=0,
    dampers=0,
    parse_springs=true, 
    parse_dampers=true,
    limits=true,
    joint_limits=Dict(vcat([[
        (Symbol(group,:_hip_joint), [-0.5,0.5]), 
        (Symbol(group,:_thigh_joint), [-0.5,1.5]), 
        (Symbol(group,:_calf_joint), [-2.5,-1])] 
        for group in [:FR, :FL, :RR, :RL]]...)),
    keep_fixed_joints=true, 
    friction_coefficient=0.8,
    contact_feet=true,
    contact_body=true,
    T=Float64)

    # mechanism
    path = joinpath(@__DIR__, "dependencies/$(string(urdf)).urdf")
    mechanism = Mechanism(path; floating=true, T,
        gravity, timestep, input_scaling, 
        parse_dampers, keep_fixed_joints)

    # springs and dampers
    !parse_springs && set_springs!(mechanism.joints, springs)
    !parse_dampers && set_dampers!(mechanism.joints, dampers)

    # joint limits
    if limits
        joints = set_limits(mechanism, joint_limits)

        mechanism = Mechanism(mechanism.origin, mechanism.bodies, joints;
            gravity, timestep, input_scaling)
    end

    # contacts
    contacts = ContactConstraint{T}[]

    if contact_feet
        # feet contacts
        body_names = [:FR_calf, :FL_calf, :RR_calf, :RL_calf]
        names = [:FR_calf_contact, :FL_calf_contact, :RR_calf_contact, :RL_calf_contact]
        contact_bodies = [get_body(mechanism, name) for name in body_names]
        n = length(contact_bodies)
        normals = fill(Z_AXIS,n)
        friction_coefficients = fill(friction_coefficient,n)
        contact_origins = fill([-0.006; 0; -0.092],n)
        contact_radii = fill(0.021,n)
        contacts = [contacts;contact_constraint(contact_bodies, normals; friction_coefficients, contact_origins, contact_radii, names)]
    end

    if contact_body
        # thigh contacts
        body_names = [:FR_thigh, :FL_thigh, :RR_thigh, :RL_thigh]
        names = [:FR_thigh_contact, :FL_thigh_contact, :RR_thigh_contact, :RL_thigh_contact]
        contact_bodies = [get_body(mechanism, name) for name in body_names]
        n = length(contact_bodies)
        normals = fill(Z_AXIS,n)
        friction_coefficients = fill(friction_coefficient,n)
        contact_origins = [
            [-0.005; -0.023; -0.16],
            [-0.005; 0.023; -0.16],
            [-0.005; -0.023; -0.16],
            [-0.005; 0.023; -0.16],
        ]
        contact_radii = fill(0.023,n)
        contacts = [contacts;contact_constraint(contact_bodies, normals; friction_coefficients, contact_origins, contact_radii, names)]
    
        # hip contacts
        body_names = [:FR_hip, :FL_hip, :RR_hip, :RL_hip]
        names = [:FR_hip_contact, :FL_hip_contact, :RR_hip_contact, :RL_hip_contact]
        contact_bodies = [get_body(mechanism, name) for name in body_names]
        n = length(contact_bodies)
        normals = fill(Z_AXIS,n)
        friction_coefficients = fill(friction_coefficient,n)
        contact_origins = fill([0; 0.05; 0],n)
        contact_radii = fill(0.05,n)
        contacts = [contacts;contact_constraint(contact_bodies, normals; friction_coefficients, contact_origins, contact_radii, names)]
    end

    mechanism = Mechanism(mechanism.origin, mechanism.bodies, mechanism.joints, contacts;
        gravity, timestep, input_scaling)

    # zero configuration
    initialize_quadruped!(mechanism)

    # construction finished
    return mechanism
end
```

```julia
function initialize_quadruped!(mechanism::Mechanism;
    body_position=[0, 0, 0], body_orientation=one(Quaternion),
    hip_angle=0, thigh_angle=pi/4, calf_angle=-pi/2)

    zero_velocities!(mechanism)
    zero_coordinates!(mechanism)

    body_position += [0, 0, 0.43]
    set_minimal_coordinates!(mechanism, get_joint(mechanism, :floating_base), [body_position; Dojo.rotation_vector(body_orientation)])
    for group in [:FR, :FL, :RR, :RL]
        set_minimal_coordinates!(mechanism, get_joint(mechanism, Symbol(group, :_hip_joint)), [hip_angle])
        set_minimal_coordinates!(mechanism, get_joint(mechanism, Symbol(group, :_thigh_joint)), [thigh_angle])
        set_minimal_coordinates!(mechanism, get_joint(mechanism, Symbol(group, :_calf_joint)), [calf_angle])
    end

    return
end

``` 

